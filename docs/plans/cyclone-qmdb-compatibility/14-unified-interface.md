# Unified Interface Design

## Principle

The user should be able to swap between MMR and balanced tree schemes with **only a type parameter change**:

```rust
// MMR scheme
let db: Qmdb<MmrScheme> = Qmdb::new(config);

// Balanced tree scheme - SAME API
let db: Qmdb<BalancedTreeScheme> = Qmdb::new(config);

// All operations are identical
db.add(entry_hash);
db.set_inactive(serial_number);
let root = db.root();
let proof = db.prove(serial_number)?;
assert!(proof.verify(&entry_hash, &root)?);
```

## Unified Traits

### StateRootScheme (Revised)

```rust
/// Configuration and behavior for a state root calculation scheme.
///
/// Implementations must provide:
/// - Hasher for computing node digests
/// - Chunk state management (bitmap + entry hashes)
/// - Proof generation and verification
/// - Root computation from chunks
///
/// All methods take `&self` to allow runtime configuration (e.g., segment size).
pub trait StateRootScheme: Send + Sync + Clone + 'static {
    /// Hash output type
    type Digest: Digest + Clone + Default + Send + Sync;

    /// Hasher implementation (reuses existing Hasher<D> trait)
    type Hasher: Hasher<Self::Digest> + Send + Sync;

    /// Per-chunk state (segment for balanced tree, bitmap chunk for MMR)
    type ChunkState: ChunkState<Digest = Self::Digest> + Send + Sync;

    /// Scheme-specific proof material (hidden from user)
    type ProofMaterial: Clone + Send + Sync + 'static;

    /// Entries per chunk - runtime configurable via scheme instance.
    /// For balanced tree: 2^config.shift() (default 2048)
    /// For MMR: configurable
    fn entries_per_chunk(&self) -> u32;

    /// Create a new hasher instance with this scheme's configuration
    fn new_hasher(&self) -> Self::Hasher;

    /// Create a new empty chunk with this scheme's configuration
    fn new_chunk(&self) -> Self::ChunkState;

    /// Compute state root from chunk roots
    fn compute_root(
        &self,
        hasher: &mut Self::Hasher,
        chunks: &[Self::ChunkState],
    ) -> Self::Digest;

    /// Generate proof material for an entry
    fn generate_proof(
        &self,
        hasher: &mut Self::Hasher,
        chunks: &[Self::ChunkState],
        serial_number: u64,
    ) -> Result<Self::ProofMaterial, SchemeError>;

    /// Verify proof material against entry and root
    fn verify_proof(
        &self,
        hasher: &mut Self::Hasher,
        material: &Self::ProofMaterial,
        entry_hash: &Self::Digest,
        expected_root: &Self::Digest,
    ) -> Result<bool, SchemeError>;

    /// Check if entry is active according to proof
    fn proof_is_active(material: &Self::ProofMaterial) -> bool;
}
```

### ChunkState (Unchanged)

```rust
/// Per-chunk state abstraction.
///
/// For balanced tree: segment with hierarchical bitmap
/// For MMR: bitmap chunk with MMR node references
pub trait ChunkState: Send + Sync + Default + Clone + 'static {
    type Digest: Digest + Clone;

    fn set_active(&mut self, index: u32) -> Result<(), SchemeError>;
    fn set_inactive(&mut self, index: u32) -> Result<(), SchemeError>;
    fn is_active(&self, index: u32) -> Result<bool, SchemeError>;
    fn set_entry_hash(&mut self, index: u32, hash: Self::Digest) -> Result<(), SchemeError>;
    fn compute_root<H: Hasher<Self::Digest>>(&mut self, hasher: &mut H) -> Result<Self::Digest, SchemeError>;
}
```

### Unified Proof Type

```rust
/// Inclusion proof for any scheme.
///
/// The internal structure varies by scheme, but the API is identical.
#[derive(Clone, Debug)]
pub struct Proof<S: StateRootScheme> {
    /// Serial number of the proven entry
    pub serial_number: u64,
    /// Scheme-specific proof material
    material: S::ProofMaterial,
}

impl<S: StateRootScheme> Proof<S> {
    /// Create a new proof (called by scheme internally)
    pub(crate) fn new(serial_number: u64, material: S::ProofMaterial) -> Self {
        Self { serial_number, material }
    }

    /// Verify this proof against an entry hash and expected root.
    /// Requires the scheme instance for configuration-aware verification.
    pub fn verify(
        &self,
        scheme: &S,
        entry_hash: &S::Digest,
        expected_root: &S::Digest,
    ) -> Result<bool, SchemeError> {
        let mut hasher = scheme.new_hasher();
        scheme.verify_proof(&mut hasher, &self.material, entry_hash, expected_root)
    }

    /// Check if the proven entry is marked as active.
    pub fn is_active(&self) -> bool {
        S::proof_is_active(&self.material)
    }

    /// Get the underlying proof material (for serialization)
    pub fn material(&self) -> &S::ProofMaterial {
        &self.material
    }
}
```

### Unified Database Interface

```rust
/// QMDB with configurable state root scheme.
///
/// The scheme is stored as an instance to support runtime configuration
/// (e.g., segment size for balanced tree scheme).
pub struct Qmdb<S: StateRootScheme> {
    scheme: S,
    hasher: S::Hasher,
    chunks: Vec<S::ChunkState>,
    entry_count: u64,
}

impl<S: StateRootScheme> Qmdb<S> {
    /// Create a new QMDB with the given scheme configuration.
    pub fn new(scheme: S) -> Self {
        Self {
            hasher: scheme.new_hasher(),
            scheme,
            chunks: Vec::new(),
            entry_count: 0,
        }
    }

    /// Get the scheme configuration.
    pub fn scheme(&self) -> &S {
        &self.scheme
    }

    /// Add an entry, returns its serial number.
    pub fn add(&mut self, entry_hash: S::Digest) -> Result<u64, SchemeError> {
        let serial_number = self.entry_count;
        let entries_per_chunk = self.scheme.entries_per_chunk() as u64;
        let chunk_id = serial_number / entries_per_chunk;
        let index_in_chunk = (serial_number % entries_per_chunk) as u32;

        // Ensure chunk exists
        while self.chunks.len() <= chunk_id as usize {
            self.chunks.push(self.scheme.new_chunk());
        }

        let chunk = &mut self.chunks[chunk_id as usize];
        chunk.set_entry_hash(index_in_chunk, entry_hash)?;
        chunk.set_active(index_in_chunk)?;

        self.entry_count += 1;
        Ok(serial_number)
    }

    /// Mark entry as inactive.
    pub fn set_inactive(&mut self, serial_number: u64) -> Result<(), SchemeError> {
        let entries_per_chunk = self.scheme.entries_per_chunk() as u64;
        let chunk_id = serial_number / entries_per_chunk;
        let index_in_chunk = (serial_number % entries_per_chunk) as u32;

        if chunk_id as usize >= self.chunks.len() {
            return Err(SchemeError::InvalidSerialNumber(serial_number));
        }

        self.chunks[chunk_id as usize].set_inactive(index_in_chunk)
    }

    /// Mark entry as active.
    pub fn set_active(&mut self, serial_number: u64) -> Result<(), SchemeError> {
        let entries_per_chunk = self.scheme.entries_per_chunk() as u64;
        let chunk_id = serial_number / entries_per_chunk;
        let index_in_chunk = (serial_number % entries_per_chunk) as u32;

        if chunk_id as usize >= self.chunks.len() {
            return Err(SchemeError::InvalidSerialNumber(serial_number));
        }

        self.chunks[chunk_id as usize].set_active(index_in_chunk)
    }

    /// Compute current state root.
    pub fn root(&mut self) -> S::Digest {
        self.scheme.compute_root(&mut self.hasher, &self.chunks)
    }

    /// Generate inclusion proof for an entry.
    pub fn prove(&mut self, serial_number: u64) -> Result<Proof<S>, SchemeError> {
        if serial_number >= self.entry_count {
            return Err(SchemeError::InvalidSerialNumber(serial_number));
        }

        let material = self.scheme.generate_proof(&mut self.hasher, &self.chunks, serial_number)?;
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
}
```

## Scheme Implementations

### BalancedTreeScheme

```rust
/// Balanced binary tree scheme with level-keyed Blake3 hashing.
///
/// Segment size is configurable at instantiation via SegmentConfig.
#[derive(Clone, Debug)]
pub struct BalancedTreeScheme {
    config: SegmentConfig,
}

impl BalancedTreeScheme {
    /// Create a new scheme with default QMDB config (2048 entries per segment).
    pub fn new() -> Self {
        Self::with_config(SegmentConfig::default_qmdb())
    }

    /// Create a new scheme with custom segment configuration.
    pub fn with_config(config: SegmentConfig) -> Self {
        Self { config }
    }

    /// Get the segment configuration.
    pub fn config(&self) -> SegmentConfig {
        self.config
    }
}

impl Default for BalancedTreeScheme {
    fn default() -> Self {
        Self::new()
    }
}

/// Proof material for balanced tree scheme.
/// Path lengths vary based on segment configuration.
#[derive(Clone, Debug)]
pub struct BalancedTreeProofMaterial {
    pub entry_index: u32,
    pub segment_id: u64,
    pub entry_path: Vec<Hash32>,   // Length = config.shift()
    pub bitmap_path: Vec<Hash32>,  // Length varies by bitmap structure
    pub active_bits_block: [u8; 32],
    pub upper_path: Vec<Hash32>,
}

impl StateRootScheme for BalancedTreeScheme {
    type Digest = Hash32;
    type Hasher = LevelKeyed;
    type ChunkState = SegmentState;
    type ProofMaterial = BalancedTreeProofMaterial;

    fn entries_per_chunk(&self) -> u32 {
        self.config.entries_per_segment()
    }

    fn new_hasher(&self) -> Self::Hasher {
        LevelKeyed::new(self.config)
    }

    fn new_chunk(&self) -> Self::ChunkState {
        SegmentState::with_config(self.config)
    }

    fn compute_root(&self, hasher: &mut Self::Hasher, chunks: &[Self::ChunkState]) -> Self::Digest {
        let mut builder = BalancedTreeRootBuilder::new(self.config, hasher.fork());
        for (id, chunk) in chunks.iter().enumerate() {
            builder.add_segment(id as u64, chunk.segment.segment_root);
        }
        builder.finalize()
    }

    fn generate_proof(
        &self,
        hasher: &mut Self::Hasher,
        chunks: &[Self::ChunkState],
        serial_number: u64,
    ) -> Result<Self::ProofMaterial, SchemeError> {
        // ... build entry_path (length = config.shift()), bitmap_path, upper_path ...
    }

    fn verify_proof(
        &self,
        hasher: &mut Self::Hasher,
        material: &Self::ProofMaterial,
        entry_hash: &Self::Digest,
        expected_root: &Self::Digest,
    ) -> Result<bool, SchemeError> {
        // Validate path lengths match config
        if material.entry_path.len() != self.config.shift() as usize {
            return Err(SchemeError::InvalidProofPathLength);
        }

        // 1. Compute entry_root from entry_hash through entry_path
        // 2. Compute bitmap_root from active_bits_block through bitmap_path
        // 3. Combine at level config.shift(): segment_root = H(entry_root, bitmap_root)
        // 4. Compute state_root through upper_path
        // 5. Compare with expected_root
    }

    fn proof_is_active(material: &Self::ProofMaterial) -> bool {
        let bit_in_block = material.entry_index % 256;
        let byte_idx = (bit_in_block / 8) as usize;
        let bit_idx = bit_in_block % 8;
        (material.active_bits_block[byte_idx] >> bit_idx) & 1 == 1
    }
}
```

### MmrScheme

```rust
/// MMR scheme with position-based hashing and grafted bitmap.
///
/// Chunk size is configurable at instantiation.
#[derive(Clone, Debug)]
pub struct MmrScheme<H: CHasher = Sha256> {
    entries_per_chunk: u32,
    _marker: PhantomData<H>,
}

impl<H: CHasher> MmrScheme<H> {
    /// Create a new scheme with default chunk size (1024).
    pub fn new() -> Self {
        Self::with_chunk_size(1024)
    }

    /// Create a new scheme with custom chunk size.
    pub fn with_chunk_size(entries_per_chunk: u32) -> Self {
        Self { entries_per_chunk, _marker: PhantomData }
    }
}

impl<H: CHasher> Default for MmrScheme<H> {
    fn default() -> Self {
        Self::new()
    }
}

/// Proof material for MMR scheme.
#[derive(Clone, Debug)]
pub struct MmrProofMaterial<D: Digest> {
    pub location: Location,
    pub chunk: Vec<u8>,
    pub mmr_proof: mmr::Proof<D>,
    pub partial_chunk_digest: Option<D>,
}

impl<H: CHasher> StateRootScheme for MmrScheme<H> {
    type Digest = H::Digest;
    type Hasher = StandardHasher<H>;
    type ChunkState = MmrChunkState<H::Digest>;
    type ProofMaterial = MmrProofMaterial<H::Digest>;

    fn entries_per_chunk(&self) -> u32 {
        self.entries_per_chunk
    }

    fn new_hasher(&self) -> Self::Hasher {
        StandardHasher::new()
    }

    fn new_chunk(&self) -> Self::ChunkState {
        MmrChunkState::default()
    }

    fn compute_root(&self, hasher: &mut Self::Hasher, chunks: &[Self::ChunkState]) -> Self::Digest {
        // Use existing grafting::Storage to compute root
        // This wraps the existing QMDB root computation logic
    }

    fn generate_proof(
        &self,
        hasher: &mut Self::Hasher,
        chunks: &[Self::ChunkState],
        serial_number: u64,
    ) -> Result<Self::ProofMaterial, SchemeError> {
        // Use existing MMR proof generation with grafted bitmap
    }

    fn verify_proof(
        &self,
        hasher: &mut Self::Hasher,
        material: &Self::ProofMaterial,
        entry_hash: &Self::Digest,
        expected_root: &Self::Digest,
    ) -> Result<bool, SchemeError> {
        // Use existing grafting::Verifier
    }

    fn proof_is_active(material: &Self::ProofMaterial) -> bool {
        // Check bit in chunk
    }
}
```

## User Experience

### Before (Scheme-Specific Code)

```rust
// BAD: Different code paths for different schemes
#[cfg(feature = "balanced_tree")]
{
    let proof: BalancedTreeProof = db.prove_balanced(sn)?;
    proof.verify_balanced(&entry, &root)?;
}

#[cfg(not(feature = "balanced_tree"))]
{
    let proof: MmrProof = db.prove_mmr(sn)?;
    proof.verify_mmr(&entry, &root)?;
}
```

### After (Unified Interface)

```rust
// GOOD: Same code for any scheme - just change the scheme instantiation
let scheme = BalancedTreeScheme::new();  // Or: BalancedTreeScheme::with_config(config)
let mut db = Qmdb::new(scheme);

db.add(entry_hash)?;

let root = db.root();
let proof = db.prove(serial_number)?;

assert!(proof.verify(db.scheme(), &entry_hash, &root)?);
assert!(proof.is_active());

// To switch schemes, only change the scheme instantiation:
// let scheme = MmrScheme::new();
// let mut db = Qmdb::new(scheme);
```

### Configuration-Based Selection

```rust
// Runtime scheme selection via enum wrapper
pub enum AnyScheme {
    Mmr(Qmdb<MmrScheme>),
    BalancedTree(Qmdb<BalancedTreeScheme>),
}

impl AnyScheme {
    pub fn new(config: &Config) -> Self {
        match config.scheme {
            SchemeType::Mmr => {
                let scheme = MmrScheme::with_chunk_size(config.chunk_size);
                AnyScheme::Mmr(Qmdb::new(scheme))
            }
            SchemeType::BalancedTree => {
                let segment_config = SegmentConfig::new(config.segment_shift)
                    .expect("invalid segment shift");
                let scheme = BalancedTreeScheme::with_config(segment_config);
                AnyScheme::BalancedTree(Qmdb::new(scheme))
            }
        }
    }
}
```

### Custom Segment Size Example

```rust
// Use non-default segment size for balanced tree
let config = SegmentConfig::new(10).unwrap(); // 1024 entries per segment
let scheme = BalancedTreeScheme::with_config(config);
let mut db = Qmdb::new(scheme);

// All operations work the same, just with different internal structure
db.add(entry_hash)?;
let root = db.root();

// Proofs will have different path lengths based on config
let proof = db.prove(0)?;
// proof.entry_path.len() == 10 (instead of 11 for default)
```

## Interface Summary

| Component | Type | User-Facing |
|-----------|------|-------------|
| `StateRootScheme` | Trait | No (implementation detail) |
| `ChunkState` | Trait | No (implementation detail) |
| `Qmdb<S>` | Struct | **Yes** |
| `Proof<S>` | Struct | **Yes** |
| `SchemeError` | Enum | **Yes** |
| `BalancedTreeScheme` | Type alias | **Yes** (configuration) |
| `MmrScheme` | Type alias | **Yes** (configuration) |

## Key Design Decisions

1. **`ProofMaterial` as associated type** - Allows scheme-specific proof internals while exposing unified `Proof<S>` API

2. **Proof generation/verification on scheme** - Keeps scheme-specific logic encapsulated

3. **Same `Proof<S>` type for all schemes** - User code doesn't care about internals

4. **Hasher reuses existing trait** - `Hasher<D>` unchanged, just different implementations

5. **ChunkState abstraction** - Segments and bitmap chunks are both "chunks" from user perspective

## Files to Update

### Task 05 (scheme-trait.md)
- Add `ProofMaterial` associated type
- Add `generate_proof()` method
- Add `verify_proof()` method
- Add `proof_is_active()` method

### Task 05b (balanced-tree-proof.md)
- Rename to just proof material, not separate proof type
- `BalancedTreeProofMaterial` instead of `BalancedTreeProof`

### New: Unified Proof Type
- `Proof<S: StateRootScheme>` wrapping material
- Same API regardless of scheme

### QMDB integration
- `Qmdb<S: StateRootScheme>` as main interface
- All methods generic over scheme
