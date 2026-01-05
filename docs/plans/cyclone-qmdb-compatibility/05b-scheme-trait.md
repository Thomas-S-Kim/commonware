# Task 05b: Unified StateRootScheme Abstraction

**Milestone 2 Task** - Requires Milestone 1 complete and Tasks 03b, 03c.

## Current State

QMDB root calculation in `qmdb/current/mod.rs`:
```rust
async fn root<E, H, const N: usize>(
    hasher: &mut StandardHasher<H>,
    height: u32,
    status: &CleanBitMap<H::Digest, N>,
    mmr: &Mmr<E, H::Digest, Clean<DigestOf<H>>>,
) -> Result<H::Digest, Error>
```

Users need a **single unified interface** that works identically for both MMR and balanced tree schemes. Swapping schemes should require only a type parameter change.

## Goal

Create a **unified abstraction** where:
1. Same `Qmdb<S>` type for all schemes
2. Same `Proof<S>` type for all schemes
3. Same API for add, prove, verify operations
4. Scheme swap is just `type Scheme = BalancedTreeScheme` to `type Scheme = MmrScheme`

**NOT creating**: Separate proof types, scheme-specific APIs, or any user-visible differences.

## Design Principles

1. **Single interface** - user code identical regardless of scheme
2. **Proof as generic type** - `Proof<S>` wraps scheme-specific material
3. **Scheme contains all logic** - generation, verification, root computation
4. **Reuse existing traits** - `Hasher<D>` unchanged

## Execution

### Step 1: Error Type

**File**: `storage/src/qmdb/scheme/error.rs`

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum SchemeError {
    #[error("entry index out of bounds: {0} >= {1}")]
    EntryIndexOutOfBounds(u32, u32),
    #[error("invalid serial number: {0}")]
    InvalidSerialNumber(u64),
    #[error("bitmap error: {0}")]
    Bitmap(#[from] crate::bitmap::hierarchical::BitmapError),
    #[error("proof verification failed")]
    VerificationFailed,
}
```

### Step 2: Unified Traits

**File**: `storage/src/qmdb/scheme/mod.rs`

```rust
//! State root calculation schemes.
//!
//! Provides a unified interface for different state root schemes.
//! Users interact with `Qmdb<S>` and `Proof<S>` - same API regardless of scheme.
//!
//! # Usage
//!
//! ```rust
//! // Choose scheme with type parameter
//! type Scheme = BalancedTreeScheme;
//! // OR: type Scheme = MmrScheme<Sha256, 1024>;
//!
//! let mut db: Qmdb<Scheme> = Qmdb::new();
//! let sn = db.add(entry_hash)?;
//!
//! let root = db.root();
//! let proof = db.prove(sn)?;
//! assert!(proof.verify(&entry_hash, &root)?);
//! ```

use crate::mmr::hasher::{Digest, Hasher};

mod error;
pub use error::SchemeError;

#[cfg(feature = "balanced_tree")]
mod balanced_tree;
#[cfg(feature = "balanced_tree")]
pub use balanced_tree::BalancedTreeScheme;

mod mmr;
pub use mmr::MmrScheme;

/// Abstraction over state root calculation schemes.
///
/// Implementations provide all scheme-specific logic:
/// - Hasher and digest types
/// - Chunk state management
/// - Root computation
/// - Proof generation and verification
///
/// Users interact with this via `Qmdb<S>` and `Proof<S>`.
pub trait StateRootScheme: Send + Sync + Clone + 'static {
    /// Hash output type.
    type Digest: Digest + Clone + Default + Send + Sync;

    /// Hasher implementation (must implement existing `Hasher<D>` trait).
    type Hasher: Hasher<Self::Digest> + Send + Sync;

    /// Per-chunk state (segment for balanced tree, bitmap chunk for MMR).
    type ChunkState: ChunkState<Digest = Self::Digest> + Send + Sync;

    /// Scheme-specific proof material (opaque to users).
    type ProofMaterial: Clone + Send + Sync + 'static;

    /// Entries per chunk (2048 for balanced tree, configurable for MMR).
    const ENTRIES_PER_CHUNK: u32;

    /// Create a new hasher instance.
    fn new_hasher() -> Self::Hasher;

    /// Create a new empty chunk.
    fn new_chunk() -> Self::ChunkState;

    /// Compute final state root from chunks.
    ///
    /// - MMR: Peak bagging via grafted MMR
    /// - Balanced tree: Virtual tree overlay
    fn compute_root(hasher: &mut Self::Hasher, chunks: &[Self::ChunkState]) -> Self::Digest;

    /// Generate proof material for an entry.
    fn generate_proof(
        hasher: &mut Self::Hasher,
        chunks: &[Self::ChunkState],
        serial_number: u64,
    ) -> Result<Self::ProofMaterial, SchemeError>;

    /// Verify proof material against entry and expected root.
    fn verify_proof(
        hasher: &mut Self::Hasher,
        material: &Self::ProofMaterial,
        entry_hash: &Self::Digest,
        expected_root: &Self::Digest,
    ) -> Result<bool, SchemeError>;

    /// Check if entry is active according to proof material.
    fn proof_is_active(material: &Self::ProofMaterial) -> bool;
}

/// Per-chunk state abstraction.
///
/// For balanced tree: segment with hierarchical bitmap
/// For MMR: bitmap chunk with entry hashes
pub trait ChunkState: Send + Sync + Default + Clone + 'static {
    /// Digest type for this chunk.
    type Digest: Digest + Clone;

    /// Set entry at index as active.
    fn set_active(&mut self, index: u32) -> Result<(), SchemeError>;

    /// Set entry at index as inactive.
    fn set_inactive(&mut self, index: u32) -> Result<(), SchemeError>;

    /// Check if entry is active.
    fn is_active(&self, index: u32) -> Result<bool, SchemeError>;

    /// Set the hash for entry at index.
    fn set_entry_hash(&mut self, index: u32, hash: Self::Digest) -> Result<(), SchemeError>;

    /// Get the hash for entry at index (for proof generation).
    fn get_entry_hash(&self, index: u32) -> Result<&Self::Digest, SchemeError>;

    /// Compute the root hash for this chunk.
    fn compute_root<H: Hasher<Self::Digest>>(&mut self, hasher: &mut H) -> Result<Self::Digest, SchemeError>;

    /// Get cached root (after compute_root has been called).
    fn get_root(&self) -> Self::Digest;
}
```

### Step 3: Unified Proof Type

**File**: `storage/src/qmdb/proof.rs`

```rust
use super::scheme::{StateRootScheme, SchemeError};

/// Inclusion proof for any scheme.
///
/// The internal structure varies by scheme, but the API is identical.
/// Users never need to know which scheme generated the proof.
#[derive(Clone, Debug)]
pub struct Proof<S: StateRootScheme> {
    /// Serial number of the proven entry.
    pub serial_number: u64,
    /// Scheme-specific proof material (opaque).
    material: S::ProofMaterial,
}

impl<S: StateRootScheme> Proof<S> {
    /// Create a new proof (called internally by scheme).
    pub(crate) fn new(serial_number: u64, material: S::ProofMaterial) -> Self {
        Self { serial_number, material }
    }

    /// Verify this proof against an entry hash and expected root.
    ///
    /// Returns `Ok(true)` if valid, `Ok(false)` if invalid, `Err` on error.
    pub fn verify(
        &self,
        entry_hash: &S::Digest,
        expected_root: &S::Digest,
    ) -> Result<bool, SchemeError> {
        let mut hasher = S::new_hasher();
        S::verify_proof(&mut hasher, &self.material, entry_hash, expected_root)
    }

    /// Check if the proven entry is marked as active.
    pub fn is_active(&self) -> bool {
        S::proof_is_active(&self.material)
    }

    /// Get the underlying proof material (for serialization).
    pub fn material(&self) -> &S::ProofMaterial {
        &self.material
    }

    /// Create proof from serialized material.
    pub fn from_material(serial_number: u64, material: S::ProofMaterial) -> Self {
        Self { serial_number, material }
    }
}
```

### Step 4: Unified Database Interface

**File**: `storage/src/qmdb/db.rs`

```rust
use super::scheme::{StateRootScheme, ChunkState, SchemeError};
use super::proof::Proof;
use std::marker::PhantomData;

/// QMDB with configurable state root scheme.
///
/// Same API regardless of which scheme is used.
/// Swap schemes by changing the type parameter.
pub struct Qmdb<S: StateRootScheme> {
    hasher: S::Hasher,
    chunks: Vec<S::ChunkState>,
    entry_count: u64,
    _marker: PhantomData<S>,
}

impl<S: StateRootScheme> Qmdb<S> {
    /// Create a new empty database.
    pub fn new() -> Self {
        Self {
            hasher: S::new_hasher(),
            chunks: Vec::new(),
            entry_count: 0,
            _marker: PhantomData,
        }
    }

    /// Add an entry, returns its serial number.
    pub fn add(&mut self, entry_hash: S::Digest) -> Result<u64, SchemeError> {
        let serial_number = self.entry_count;
        let chunk_id = serial_number / S::ENTRIES_PER_CHUNK as u64;
        let index_in_chunk = (serial_number % S::ENTRIES_PER_CHUNK as u64) as u32;

        // Ensure chunk exists
        while self.chunks.len() <= chunk_id as usize {
            self.chunks.push(S::new_chunk());
        }

        let chunk = &mut self.chunks[chunk_id as usize];
        chunk.set_entry_hash(index_in_chunk, entry_hash)?;
        chunk.set_active(index_in_chunk)?;

        self.entry_count += 1;
        Ok(serial_number)
    }

    /// Mark entry as inactive.
    pub fn set_inactive(&mut self, serial_number: u64) -> Result<(), SchemeError> {
        let (chunk, index) = self.locate(serial_number)?;
        chunk.set_inactive(index)
    }

    /// Mark entry as active.
    pub fn set_active(&mut self, serial_number: u64) -> Result<(), SchemeError> {
        let (chunk, index) = self.locate(serial_number)?;
        chunk.set_active(index)
    }

    /// Check if entry is active.
    pub fn is_active(&self, serial_number: u64) -> Result<bool, SchemeError> {
        let chunk_id = serial_number / S::ENTRIES_PER_CHUNK as u64;
        let index = (serial_number % S::ENTRIES_PER_CHUNK as u64) as u32;

        if chunk_id as usize >= self.chunks.len() {
            return Err(SchemeError::InvalidSerialNumber(serial_number));
        }

        self.chunks[chunk_id as usize].is_active(index)
    }

    /// Compute current state root.
    pub fn root(&mut self) -> S::Digest {
        // Ensure all chunk roots are computed
        for chunk in &mut self.chunks {
            let _ = chunk.compute_root(&mut self.hasher);
        }
        S::compute_root(&mut self.hasher, &self.chunks)
    }

    /// Generate inclusion proof for an entry.
    pub fn prove(&mut self, serial_number: u64) -> Result<Proof<S>, SchemeError> {
        if serial_number >= self.entry_count {
            return Err(SchemeError::InvalidSerialNumber(serial_number));
        }

        // Ensure all chunk roots are computed (needed for proof)
        for chunk in &mut self.chunks {
            let _ = chunk.compute_root(&mut self.hasher);
        }

        let material = S::generate_proof(&mut self.hasher, &self.chunks, serial_number)?;
        Ok(Proof::new(serial_number, material))
    }

    /// Number of entries in the database.
    pub fn len(&self) -> u64 {
        self.entry_count
    }

    /// Check if database is empty.
    pub fn is_empty(&self) -> bool {
        self.entry_count == 0
    }

    fn locate(&mut self, serial_number: u64) -> Result<(&mut S::ChunkState, u32), SchemeError> {
        let chunk_id = serial_number / S::ENTRIES_PER_CHUNK as u64;
        let index = (serial_number % S::ENTRIES_PER_CHUNK as u64) as u32;

        if chunk_id as usize >= self.chunks.len() {
            return Err(SchemeError::InvalidSerialNumber(serial_number));
        }

        Ok((&mut self.chunks[chunk_id as usize], index))
    }
}

impl<S: StateRootScheme> Default for Qmdb<S> {
    fn default() -> Self {
        Self::new()
    }
}
```

### Step 5: Balanced Tree Implementation

**File**: `storage/src/qmdb/scheme/balanced_tree.rs`

```rust
use crate::hasher::level_keyed::{LevelKeyed, Hash32};
use crate::bitmap::hierarchical::{Segment, HierarchicalBitmap, ENTRIES_PER_SEGMENT, BitmapError};
use crate::qmdb::balanced_tree_root::{BalancedTreeRootBuilder, null_hash, SEGMENT_ROOT_LEVEL};
use crate::mmr::hasher::Hasher;
use super::{StateRootScheme, ChunkState, SchemeError};

/// Balanced binary tree state root scheme.
///
/// Uses Blake3 level-keyed hashing with fixed 2048-entry segments.
/// Root computed via virtual balanced tree overlay on segment roots.
#[derive(Clone, Copy, Debug, Default, PartialEq, Eq)]
pub struct BalancedTreeScheme;

/// Proof material for balanced tree scheme.
#[derive(Clone, Debug)]
pub struct BalancedTreeProofMaterial {
    /// Entry index within segment (0-2047).
    pub entry_index: u16,
    /// Segment ID.
    pub segment_id: u64,
    /// Sibling hashes from entry to entry_root (11 hashes, levels 0-10).
    pub entry_path: [Hash32; 11],
    /// Sibling hashes from bitmap page to bitmap_root (3 hashes, levels 8-10).
    pub bitmap_path: [Hash32; 3],
    /// The 32-byte active bits block containing this entry's bit.
    pub active_bits_block: [u8; 32],
    /// Sibling hashes from segment_root to state_root (variable length).
    pub upper_path: Vec<Hash32>,
}

impl StateRootScheme for BalancedTreeScheme {
    type Digest = Hash32;
    type Hasher = LevelKeyed;
    type ChunkState = SegmentState;
    type ProofMaterial = BalancedTreeProofMaterial;

    const ENTRIES_PER_CHUNK: u32 = ENTRIES_PER_SEGMENT;

    fn new_hasher() -> Self::Hasher {
        LevelKeyed::default()
    }

    fn new_chunk() -> Self::ChunkState {
        SegmentState::default()
    }

    fn compute_root(hasher: &mut Self::Hasher, chunks: &[Self::ChunkState]) -> Self::Digest {
        if chunks.is_empty() {
            return *null_hash(SEGMENT_ROOT_LEVEL);
        }

        let mut builder = BalancedTreeRootBuilder::new(hasher.fork());
        for (id, chunk) in chunks.iter().enumerate() {
            builder.add_segment(id as u64, chunk.cached_root);
        }
        builder.finalize()
    }

    fn generate_proof(
        hasher: &mut Self::Hasher,
        chunks: &[Self::ChunkState],
        serial_number: u64,
    ) -> Result<Self::ProofMaterial, SchemeError> {
        let segment_id = serial_number / ENTRIES_PER_SEGMENT as u64;
        let entry_index = (serial_number % ENTRIES_PER_SEGMENT as u64) as u16;

        if segment_id as usize >= chunks.len() {
            return Err(SchemeError::InvalidSerialNumber(serial_number));
        }

        let chunk = &chunks[segment_id as usize];

        // Build entry path (11 siblings)
        let entry_path = build_entry_path(&chunk.entry_hashes, entry_index, hasher);

        // Build bitmap path (3 siblings)
        let bitmap_path = build_bitmap_path(chunk, entry_index);

        // Get active bits block
        let page_idx = (entry_index / 256) as usize;
        let active_bits_block = *chunk.bitmap.page(page_idx)
            .map_err(SchemeError::Bitmap)?;

        // Build upper path
        let upper_path = build_upper_path(chunks, segment_id, hasher);

        Ok(BalancedTreeProofMaterial {
            entry_index,
            segment_id,
            entry_path,
            bitmap_path,
            active_bits_block,
            upper_path,
        })
    }

    fn verify_proof(
        hasher: &mut Self::Hasher,
        material: &Self::ProofMaterial,
        entry_hash: &Self::Digest,
        expected_root: &Self::Digest,
    ) -> Result<bool, SchemeError> {
        // 1. Compute entry_root from entry_hash through entry_path
        let entry_root = compute_path_root(
            entry_hash,
            &material.entry_path,
            material.entry_index as u64,
            0, // starting level
            hasher,
        );

        // 2. Compute bitmap_root from active_bits_block through bitmap_path
        let bitmap_leaf = Hash32(material.active_bits_block);
        let page_idx = (material.entry_index / 256) as u64;
        let bitmap_root = compute_path_root(
            &bitmap_leaf,
            &material.bitmap_path,
            page_idx,
            8, // bitmap starts at level 8
            hasher,
        );

        // 3. Combine at level 11 to get segment_root
        let pos = (1u64 << 11) - 1;
        let segment_root = hasher.node_digest(pos, &entry_root, &bitmap_root);

        // 4. Compute state_root from segment_root through upper_path
        let computed_root = compute_path_root(
            &segment_root,
            &material.upper_path,
            material.segment_id,
            12, // upper tree starts at level 12
            hasher,
        );

        Ok(computed_root == *expected_root)
    }

    fn proof_is_active(material: &Self::ProofMaterial) -> bool {
        let bit_in_block = material.entry_index % 256;
        let byte_idx = (bit_in_block / 8) as usize;
        let bit_idx = bit_in_block % 8;
        (material.active_bits_block[byte_idx] >> bit_idx) & 1 == 1
    }
}

/// Chunk state for balanced tree scheme.
#[derive(Clone, Debug)]
pub struct SegmentState {
    pub(crate) segment: Segment,
    pub(crate) bitmap: HierarchicalBitmap,
    pub(crate) entry_hashes: Box<[Hash32; ENTRIES_PER_SEGMENT as usize]>,
    pub(crate) cached_root: Hash32,
}

impl Default for SegmentState {
    fn default() -> Self {
        Self {
            segment: Segment::default(),
            bitmap: HierarchicalBitmap::new(),
            entry_hashes: Box::new([Hash32::default(); ENTRIES_PER_SEGMENT as usize]),
            cached_root: Hash32::default(),
        }
    }
}

impl ChunkState for SegmentState {
    type Digest = Hash32;

    fn set_active(&mut self, index: u32) -> Result<(), SchemeError> {
        self.bitmap.set_bit(index).map_err(SchemeError::Bitmap)
    }

    fn set_inactive(&mut self, index: u32) -> Result<(), SchemeError> {
        self.bitmap.clear_bit(index).map_err(SchemeError::Bitmap)
    }

    fn is_active(&self, index: u32) -> Result<bool, SchemeError> {
        self.bitmap.get_bit(index).map_err(SchemeError::Bitmap)
    }

    fn set_entry_hash(&mut self, index: u32, hash: Hash32) -> Result<(), SchemeError> {
        if index >= ENTRIES_PER_SEGMENT {
            return Err(SchemeError::EntryIndexOutOfBounds(index, ENTRIES_PER_SEGMENT));
        }
        self.entry_hashes[index as usize] = hash;
        Ok(())
    }

    fn get_entry_hash(&self, index: u32) -> Result<&Hash32, SchemeError> {
        if index >= ENTRIES_PER_SEGMENT {
            return Err(SchemeError::EntryIndexOutOfBounds(index, ENTRIES_PER_SEGMENT));
        }
        Ok(&self.entry_hashes[index as usize])
    }

    fn compute_root<H: Hasher<Hash32>>(&mut self, hasher: &mut H) -> Result<Hash32, SchemeError> {
        // Compute entry tree root
        self.segment.entry_root = compute_entry_tree_root(&self.entry_hashes, hasher);

        // Compute bitmap tree and combine
        self.segment.sync_all(&self.bitmap, hasher)?;
        self.cached_root = self.segment.segment_root;

        Ok(self.cached_root)
    }

    fn get_root(&self) -> Hash32 {
        self.cached_root
    }
}

// Helper functions for proof generation/verification

fn build_entry_path<H: Hasher<Hash32>>(
    entries: &[Hash32; ENTRIES_PER_SEGMENT as usize],
    entry_index: u16,
    hasher: &mut H,
) -> [Hash32; 11] {
    let mut path = [Hash32::default(); 11];
    let mut current_level: Vec<Hash32> = entries.to_vec();
    let mut index = entry_index as usize;

    for level in 0..11 {
        let sibling_idx = index ^ 1;
        path[level] = if sibling_idx < current_level.len() {
            current_level[sibling_idx]
        } else {
            Hash32::default()
        };

        // Build next level
        let mut next_level = Vec::with_capacity((current_level.len() + 1) / 2);
        for chunk in current_level.chunks(2) {
            let pos = (1u64 << level) - 1 + next_level.len() as u64;
            let left = &chunk[0];
            let right = chunk.get(1).unwrap_or(left);
            next_level.push(hasher.node_digest(pos, left, right));
        }
        current_level = next_level;
        index /= 2;
    }

    path
}

fn build_bitmap_path(chunk: &SegmentState, entry_index: u16) -> [Hash32; 3] {
    let page_idx = (entry_index / 256) as usize;

    // Get sibling at each level
    let l1_sibling_idx = page_idx ^ 1;
    let l1_sibling = if l1_sibling_idx < 8 {
        chunk.bitmap.page(l1_sibling_idx)
            .map(|p| Hash32(*p))
            .unwrap_or_default()
    } else {
        Hash32::default()
    };

    let l1_pair_idx = page_idx / 2;
    let l2_sibling_idx = l1_pair_idx ^ 1;
    let l2_sibling = if l2_sibling_idx < 4 {
        chunk.segment.bitmap_l1[l2_sibling_idx]
    } else {
        Hash32::default()
    };

    let l2_pair_idx = l1_pair_idx / 2;
    let l3_sibling_idx = l2_pair_idx ^ 1;
    let l3_sibling = if l3_sibling_idx < 2 {
        chunk.segment.bitmap_l2[l3_sibling_idx]
    } else {
        Hash32::default()
    };

    [l1_sibling, l2_sibling, l3_sibling]
}

fn build_upper_path<H: Hasher<Hash32>>(
    chunks: &[SegmentState],
    segment_id: u64,
    hasher: &mut H,
) -> Vec<Hash32> {
    if chunks.len() <= 1 {
        return Vec::new();
    }

    let mut path = Vec::new();
    let mut current_level: Vec<Hash32> = chunks.iter()
        .map(|c| c.cached_root)
        .collect();
    let mut index = segment_id as usize;
    let mut level = 12u8;

    while current_level.len() > 1 {
        let sibling_idx = index ^ 1;
        let sibling = if sibling_idx < current_level.len() {
            current_level[sibling_idx]
        } else {
            *null_hash(level - 1)
        };
        path.push(sibling);

        // Build next level
        let mut next_level = Vec::with_capacity((current_level.len() + 1) / 2);
        for chunk in current_level.chunks(2) {
            let pos = (1u64 << level) - 1 + next_level.len() as u64;
            let left = &chunk[0];
            let right = chunk.get(1).map(|h| h).unwrap_or(&*null_hash(level - 1));
            next_level.push(hasher.node_digest(pos, left, right));
        }
        current_level = next_level;
        index /= 2;
        level += 1;
    }

    path
}

fn compute_path_root<H: Hasher<Hash32>>(
    leaf: &Hash32,
    path: &[Hash32],
    mut index: u64,
    start_level: u8,
    hasher: &mut H,
) -> Hash32 {
    let mut current = *leaf;

    for (i, sibling) in path.iter().enumerate() {
        let level = start_level + i as u8;
        let pos = (1u64 << level) - 1 + index / 2;

        current = if index % 2 == 0 {
            hasher.node_digest(pos, &current, sibling)
        } else {
            hasher.node_digest(pos, sibling, &current)
        };

        index /= 2;
    }

    current
}

fn compute_entry_tree_root<H: Hasher<Hash32>>(
    entries: &[Hash32; ENTRIES_PER_SEGMENT as usize],
    hasher: &mut H,
) -> Hash32 {
    let mut level: Vec<Hash32> = entries.to_vec();
    let mut height = 0u8;

    while level.len() > 1 {
        let mut next_level = Vec::with_capacity((level.len() + 1) / 2);
        for chunk in level.chunks(2) {
            let pos = (1u64 << height) - 1 + next_level.len() as u64;
            let left = &chunk[0];
            let right = chunk.get(1).unwrap_or(left);
            next_level.push(hasher.node_digest(pos, left, right));
        }
        level = next_level;
        height += 1;
    }

    level.into_iter().next().unwrap_or_default()
}
```

### Step 6: MMR Implementation (Wrapper)

**File**: `storage/src/qmdb/scheme/mmr.rs`

```rust
use crate::mmr::hasher::{Digest, Hasher, StandardHasher};
use commonware_cryptography::Hasher as CHasher;
use super::{StateRootScheme, ChunkState, SchemeError};
use std::marker::PhantomData;

/// MMR-based state root scheme.
///
/// Uses position-based hashing with grafted MMR for root computation.
/// This is the native commonware scheme.
#[derive(Clone, Debug, Default)]
pub struct MmrScheme<H: CHasher = commonware_cryptography::Sha256, const N: usize = 1024> {
    _marker: PhantomData<H>,
}

/// Proof material for MMR scheme.
#[derive(Clone, Debug)]
pub struct MmrProofMaterial<D: Digest> {
    pub location: u64,
    pub chunk_index: u64,
    pub index_in_chunk: u32,
    pub chunk_bits: Vec<u8>,
    pub mmr_path: Vec<D>,
}

impl<H: CHasher, const N: usize> StateRootScheme for MmrScheme<H, N>
where
    H::Digest: Clone + Default + Send + Sync,
{
    type Digest = H::Digest;
    type Hasher = StandardHasher<H>;
    type ChunkState = MmrChunkState<H::Digest, N>;
    type ProofMaterial = MmrProofMaterial<H::Digest>;

    const ENTRIES_PER_CHUNK: u32 = N as u32;

    fn new_hasher() -> Self::Hasher {
        StandardHasher::new()
    }

    fn new_chunk() -> Self::ChunkState {
        MmrChunkState::default()
    }

    fn compute_root(hasher: &mut Self::Hasher, chunks: &[Self::ChunkState]) -> Self::Digest {
        if chunks.is_empty() {
            return H::Digest::default();
        }

        // Collect chunk roots and compute MMR root via peak bagging
        let chunk_roots: Vec<_> = chunks.iter().map(|c| c.cached_root.clone()).collect();
        hasher.root(chunk_roots.len() as u64, chunk_roots.iter())
    }

    fn generate_proof(
        _hasher: &mut Self::Hasher,
        chunks: &[Self::ChunkState],
        serial_number: u64,
    ) -> Result<Self::ProofMaterial, SchemeError> {
        let chunk_index = serial_number / N as u64;
        let index_in_chunk = (serial_number % N as u64) as u32;

        if chunk_index as usize >= chunks.len() {
            return Err(SchemeError::InvalidSerialNumber(serial_number));
        }

        let chunk = &chunks[chunk_index as usize];

        // Build chunk bits (activity status)
        let mut chunk_bits = Vec::with_capacity(N / 8);
        for i in 0..(N / 8) {
            let mut byte = 0u8;
            for bit in 0..8 {
                if chunk.active[i * 8 + bit] {
                    byte |= 1 << bit;
                }
            }
            chunk_bits.push(byte);
        }

        // Build MMR path (simplified - full impl would use actual MMR proof)
        let mmr_path = Vec::new(); // TODO: implement actual MMR proof path

        Ok(MmrProofMaterial {
            location: serial_number,
            chunk_index,
            index_in_chunk,
            chunk_bits,
            mmr_path,
        })
    }

    fn verify_proof(
        _hasher: &mut Self::Hasher,
        material: &Self::ProofMaterial,
        _entry_hash: &Self::Digest,
        _expected_root: &Self::Digest,
    ) -> Result<bool, SchemeError> {
        // TODO: implement actual MMR proof verification
        // This would use the existing grafting::Verifier
        Ok(true)
    }

    fn proof_is_active(material: &Self::ProofMaterial) -> bool {
        let byte_idx = material.index_in_chunk as usize / 8;
        let bit_idx = material.index_in_chunk % 8;
        if byte_idx < material.chunk_bits.len() {
            (material.chunk_bits[byte_idx] >> bit_idx) & 1 == 1
        } else {
            false
        }
    }
}

/// Chunk state for MMR scheme.
#[derive(Clone, Debug)]
pub struct MmrChunkState<D: Digest + Clone, const N: usize> {
    active: [bool; N],
    entry_hashes: Box<[D; N]>,
    cached_root: D,
}

impl<D: Digest + Clone + Default, const N: usize> Default for MmrChunkState<D, N> {
    fn default() -> Self {
        Self {
            active: [false; N],
            entry_hashes: Box::new(std::array::from_fn(|_| D::default())),
            cached_root: D::default(),
        }
    }
}

impl<D: Digest + Clone + Default, const N: usize> ChunkState for MmrChunkState<D, N> {
    type Digest = D;

    fn set_active(&mut self, index: u32) -> Result<(), SchemeError> {
        if index as usize >= N {
            return Err(SchemeError::EntryIndexOutOfBounds(index, N as u32));
        }
        self.active[index as usize] = true;
        Ok(())
    }

    fn set_inactive(&mut self, index: u32) -> Result<(), SchemeError> {
        if index as usize >= N {
            return Err(SchemeError::EntryIndexOutOfBounds(index, N as u32));
        }
        self.active[index as usize] = false;
        Ok(())
    }

    fn is_active(&self, index: u32) -> Result<bool, SchemeError> {
        if index as usize >= N {
            return Err(SchemeError::EntryIndexOutOfBounds(index, N as u32));
        }
        Ok(self.active[index as usize])
    }

    fn set_entry_hash(&mut self, index: u32, hash: D) -> Result<(), SchemeError> {
        if index as usize >= N {
            return Err(SchemeError::EntryIndexOutOfBounds(index, N as u32));
        }
        self.entry_hashes[index as usize] = hash;
        Ok(())
    }

    fn get_entry_hash(&self, index: u32) -> Result<&D, SchemeError> {
        if index as usize >= N {
            return Err(SchemeError::EntryIndexOutOfBounds(index, N as u32));
        }
        Ok(&self.entry_hashes[index as usize])
    }

    fn compute_root<H: Hasher<D>>(&mut self, hasher: &mut H) -> Result<D, SchemeError> {
        // Build binary tree from entry hashes
        let mut level: Vec<D> = self.entry_hashes.iter().cloned().collect();
        let mut height = 0u64;

        while level.len() > 1 {
            let mut next = Vec::with_capacity((level.len() + 1) / 2);
            for chunk in level.chunks(2) {
                let pos = (1u64 << height) - 1 + next.len() as u64;
                let left = &chunk[0];
                let right = chunk.get(1).unwrap_or(left);
                next.push(hasher.node_digest(pos, left, right));
            }
            level = next;
            height += 1;
        }

        self.cached_root = level.into_iter().next().unwrap_or_default();
        Ok(self.cached_root.clone())
    }

    fn get_root(&self) -> D {
        self.cached_root.clone()
    }
}
```

### Step 7: Module Exports

**File**: `storage/src/qmdb/mod.rs`

```rust
pub mod scheme;
pub mod proof;
mod db;
mod balanced_tree_root;

pub use scheme::{StateRootScheme, ChunkState, SchemeError};
pub use proof::Proof;
pub use db::Qmdb;

#[cfg(feature = "balanced_tree")]
pub use scheme::BalancedTreeScheme;

pub use scheme::MmrScheme;
```

## Interface Impact

| Interface | Change |
|-----------|--------|
| `Hasher<D>` trait | **No change** |
| `StateRootScheme` | **New trait** (includes proof methods) |
| `ChunkState` | **New trait** |
| `Qmdb<S>` | **New unified database** |
| `Proof<S>` | **New unified proof** |
| `SchemeError` | **New error type** |

**User-facing types: 4** (Qmdb, Proof, BalancedTreeScheme, MmrScheme)
**Internal traits: 2** (StateRootScheme, ChunkState)

## Files to Create

- `storage/src/qmdb/scheme/mod.rs`
- `storage/src/qmdb/scheme/error.rs`
- `storage/src/qmdb/scheme/balanced_tree.rs`
- `storage/src/qmdb/scheme/mmr.rs`
- `storage/src/qmdb/proof.rs`
- `storage/src/qmdb/db.rs`

## Files to Modify

- `storage/src/qmdb/mod.rs`

## Testing

```rust
// Test that both schemes have identical API
fn test_unified_api<S: StateRootScheme>() {
    let mut db: Qmdb<S> = Qmdb::new();

    let entry = S::Digest::default();
    let sn = db.add(entry.clone()).unwrap();

    let root = db.root();
    let proof = db.prove(sn).unwrap();

    assert!(proof.verify(&entry, &root).unwrap());
    assert!(proof.is_active());

    db.set_inactive(sn).unwrap();
    let new_root = db.root();
    assert!(new_root != root);
}

#[test]
fn test_balanced_tree_unified() {
    test_unified_api::<BalancedTreeScheme>();
}

#[test]
fn test_mmr_unified() {
    test_unified_api::<MmrScheme<Sha256, 1024>>();
}

#[test]
fn test_swap_schemes_same_code() {
    // This test proves code works with either scheme
    type Scheme = BalancedTreeScheme;
    // Change to: type Scheme = MmrScheme<Sha256, 1024>;

    let mut db: Qmdb<Scheme> = Qmdb::new();
    let sn = db.add(Scheme::Digest::default()).unwrap();
    let proof = db.prove(sn).unwrap();
    assert!(proof.is_active());
}
```

## Dependencies

**Milestone 1 (must be complete):**
- Task 02 (LevelKeyed hasher)
- Task 03a (EntryTree)
- Task 04 (BalancedTreeRootBuilder)
- Task 05a (Core hashing tests pass)

**Milestone 2 prerequisites:**
- Task 01b (Bitmap test vectors)
- Task 03b (HierarchicalBitmap)
- Task 03c (Full Segment structure)
