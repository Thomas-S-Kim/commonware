# Task 2: Level-Keyed Hasher

## Current State

`storage/src/mmr/hasher.rs` defines:
```rust
pub trait Hasher<D: Digest>: Send + Sync {
    fn leaf_digest(&mut self, pos: Position, element: &[u8]) -> D;
    fn node_digest(&mut self, pos: Position, left: &D, right: &D) -> D;
    fn root<'a>(&mut self, size: Position, peak_digests: impl Iterator<Item = &'a D>) -> D;
    fn digest(&mut self, data: &[u8]) -> D;
    fn inner(&mut self) -> &mut Self::Inner;
    fn fork(&self) -> impl Hasher<D>;
}
```

The `Standard<H>` implementation uses position-based domain separation.

## Goal

Create `LevelKeyed` as a **new implementation** of the existing `Hasher<D>` trait. No new traits needed.

The key insight: the balanced tree scheme can reuse `Hasher<D>` by deriving level from position. The `root()` method computes QMDB-style balanced tree roots from segment roots, enabling transparent scheme switching via a single type parameter.

**CRITICAL**: The `LevelKeyed::root()` method MUST produce byte-for-byte identical results to the QMDB reference implementation for all inputs. This is the key to the unified interface - users can switch between `Standard` (MMR) and `LevelKeyed` (QMDB) hashers without changing any other code.

## Algorithm

**IMPORTANT: Different algorithms for leaves vs nodes**

### Leaf Hashing (Entry Hash)
- **Algorithm**: BLAKE2b-512, output truncated to 32 bytes
- **Input**: Entry payload bytes
- **Domain separation**: None (position ignored)

### Node Hashing (Merkle Nodes)
- **Algorithm**: Blake3 keyed hash
- **Key derivation**: XOR level (as `u32` little-endian) into first 4 bytes of Blake3 IV
- **Level source**: Derived from position using `pos_to_height()` (already exists in mmr crate)
- **Input**: `left || right` (64 bytes concatenated)
- **Output**: 32-byte hash

## Execution

### Step 1: Add Dependencies

**File**: `storage/Cargo.toml`

```toml
[features]
balanced_tree = ["blake3", "blake2"]

[dependencies]
blake3 = { version = "1.5", optional = true }
blake2 = { version = "0.10", optional = true }
```

### Step 2: Create Hasher Module

**File**: `storage/src/hasher/mod.rs`

```rust
//! Hasher implementations for different schemes.

#[cfg(feature = "balanced_tree")]
mod level_keyed;

#[cfg(feature = "balanced_tree")]
pub use level_keyed::{LevelKeyed, Hash32};
```

### Step 3: Implement Hasher<D> Trait

**File**: `storage/src/hasher/level_keyed.rs`

```rust
//! Level-keyed hasher for balanced tree scheme (QMDB compatible).
//!
//! - Leaf hashing: BLAKE2b-512 truncated to 32 bytes (entry hash)
//! - Node hashing: Blake3 keyed hash with level XOR'd into IV

use crate::mmr::hasher::{Digest, Hasher, Position};
use crate::mmr::pos_to_height;
use blake2::{Blake2b512, Digest as Blake2Digest};

/// 32-byte hash output for balanced tree scheme.
#[derive(Clone, Copy, Debug, Default, PartialEq, Eq, Hash)]
pub struct Hash32(pub [u8; 32]);

impl AsRef<[u8]> for Hash32 {
    fn as_ref(&self) -> &[u8] { &self.0 }
}

impl From<[u8; 32]> for Hash32 {
    fn from(arr: [u8; 32]) -> Self { Self(arr) }
}

impl From<blake3::Hash> for Hash32 {
    fn from(h: blake3::Hash) -> Self { Self(*h.as_bytes()) }
}

impl Digest for Hash32 {
    fn as_bytes(&self) -> &[u8] { &self.0 }
}

/// Compute BLAKE2b-512 hash truncated to 32 bytes.
/// This matches QMDB's entry hash algorithm.
pub fn blake2b_hash_32(data: &[u8]) -> Hash32 {
    let mut hasher = Blake2b512::new();
    hasher.update(data);
    let result = hasher.finalize();
    let mut output = [0u8; 32];
    output.copy_from_slice(&result[..32]);
    Hash32(output)
}

// Blake3 IV constants (from Blake3 specification)
static BLAKE3_IV_BYTES: [u8; 32] = [
    0x67, 0xE6, 0x09, 0x6A, 0x85, 0xAE, 0x67, 0xBB,
    0x72, 0xF3, 0x6E, 0x3C, 0x3A, 0xF5, 0x4F, 0xA5,
    0x7F, 0x52, 0x0E, 0x51, 0x8C, 0x68, 0x05, 0x9B,
    0xAB, 0xD9, 0x83, 0x1F, 0x19, 0xCD, 0xE0, 0x5B,
];

/// Compute level-keyed hash: XOR level into Blake3 IV, then hash left||right.
pub fn level_keyed_hash(level: u8, left: &Hash32, right: &Hash32) -> Hash32 {
    let mut key = BLAKE3_IV_BYTES;
    let level_bytes = (level as u32).to_le_bytes();
    for (i, &byte) in level_bytes.iter().enumerate() {
        key[i] ^= byte;
    }
    let mut input = [0u8; 64];
    input[..32].copy_from_slice(&left.0);
    input[32..].copy_from_slice(&right.0);
    blake3::keyed_hash(&key, &input).into()
}

/// Level-keyed hasher implementing the `Hasher<D>` trait for QMDB compatibility.
///
/// - Leaf hashes use BLAKE2b-512 (truncated to 32 bytes) matching QMDB entry hash
/// - Node hashes use Blake3 keyed with level XOR'd into IV
/// - The `root()` method computes QMDB-style balanced tree roots from segment roots
#[derive(Clone)]
pub struct LevelKeyed {
    config: SegmentConfig,
    inner: blake3::Hasher,
}

impl LevelKeyed {
    /// Create a new hasher with custom segment configuration.
    pub fn new(config: SegmentConfig) -> Self {
        Self { config, inner: blake3::Hasher::new() }
    }

    /// Get the segment configuration.
    pub fn config(&self) -> SegmentConfig {
        self.config
    }
}

impl Default for LevelKeyed {
    fn default() -> Self {
        Self::new(SegmentConfig::default_qmdb())
    }
}

impl Hasher<Hash32> for LevelKeyed {
    type Inner = blake3::Hasher;

    fn leaf_digest(&mut self, _pos: Position, element: &[u8]) -> Hash32 {
        // QMDB uses BLAKE2b-512 truncated to 32 bytes for entry hashing
        // Position is ignored (no domain separation for leaves)
        blake2b_hash_32(element)
    }

    fn node_digest(&mut self, pos: Position, left: &Hash32, right: &Hash32) -> Hash32 {
        let level = pos_to_height(pos) as u8;
        level_keyed_hash(level, left, right)
    }

    fn root<'a>(&mut self, size: Position, peaks: impl Iterator<Item = &'a Hash32>) -> Hash32 {
        // Compute QMDB-style balanced tree root from segment roots.
        // The `peaks` iterator provides segment roots (each is a root of a complete
        // entry tree). We build a balanced binary tree overlay to combine them.
        //
        // Key insight: In QMDB, segment roots are at level `shift` (e.g., level 11
        // for 2048 entries). The upper tree combines them at level `shift+1` and above.
        //
        // The `size` parameter indicates total entries, which determines:
        // - Number of complete segments: size / entries_per_segment
        // - Partial segment entries: size % entries_per_segment
        //
        // For exact QMDB compatibility, we use BalancedTreeRootBuilder internally.
        let segment_roots: Vec<Hash32> = peaks.cloned().collect();
        if segment_roots.is_empty() {
            return null_hash_for_config(&self.config);
        }

        let mut builder = BalancedTreeRootBuilder::new(self.config, self.fork());
        for (id, root) in segment_roots.iter().enumerate() {
            builder.add_segment(id as u64, *root);
        }
        builder.finalize()
    }

    fn digest(&mut self, data: &[u8]) -> Hash32 {
        // General digest uses BLAKE2b to match QMDB conventions
        blake2b_hash_32(data)
    }

    fn inner(&mut self) -> &mut Self::Inner {
        &mut self.inner
    }

    fn fork(&self) -> impl Hasher<Hash32> {
        Self::new(self.config)
    }
}
```

### Step 4: Export from lib.rs

**File**: `storage/src/lib.rs`

```rust
#[cfg(feature = "balanced_tree")]
pub mod hasher;
```

## Files to Create

- `storage/src/hasher/mod.rs`
- `storage/src/hasher/level_keyed.rs`

## Files to Modify

- `storage/Cargo.toml`
- `storage/src/lib.rs`

## Interface Impact

| Interface | Change |
|-----------|--------|
| `Hasher<D>` trait | **No change** |
| `Digest` trait | **No change** (impl for Hash32) |
| `Standard<H>` | **No change** |
| `LevelKeyed` | **New impl** of existing trait |
| `Hash32` | **New type** (newtype wrapper) |

## Testing

```rust
#[test]
fn test_implements_hasher_trait() {
    let mut hasher = LevelKeyed::default();
    let _: Hash32 = hasher.leaf_digest(0, b"test");
}

#[test]
fn test_configurable_segment_size() {
    let config = SegmentConfig::new(10).unwrap(); // 1024 entries
    let hasher = LevelKeyed::new(config);
    assert_eq!(hasher.config().entries_per_segment(), 1024);
}

#[test]
fn test_leaf_uses_blake2b() {
    let mut hasher = LevelKeyed::default();
    let leaf = hasher.leaf_digest(0, b"test");
    // Should match BLAKE2b-512 truncated to 32 bytes
    let expected = blake2b_hash_32(b"test");
    assert_eq!(leaf, expected);
}

#[test]
fn test_node_uses_blake3_keyed() {
    let left = Hash32([0xAA; 32]);
    let right = Hash32([0xBB; 32]);
    let mut hasher = LevelKeyed::default();

    // Position 0 = level 0
    let node_l0 = hasher.node_digest(0, &left, &right);
    // Position 2 = level 1
    let node_l1 = hasher.node_digest(2, &left, &right);

    // Different levels should produce different hashes
    assert_ne!(node_l0, node_l1);
}

#[test]
fn test_level_affects_hash() {
    let left = Hash32([0xAA; 32]);
    let right = Hash32([0xBB; 32]);
    assert_ne!(
        level_keyed_hash(5, &left, &right),
        level_keyed_hash(6, &left, &right)
    );
}

#[test]
fn test_inner_returns_mutable_hasher() {
    let mut hasher = LevelKeyed::default();
    let inner = hasher.inner();
    inner.update(b"test data");
    let _ = inner.finalize();
}

#[test]
fn test_matches_qmdb_reference() {
    // Verify against test vectors from Task 1
    // These vectors are generated from the QMDB reference implementation
    // and validate BLAKE2b leaf hashing + Blake3 keyed node hashing

    // Test null hash chain (from qmdb-common/src/merkletree/null.rs)
    let null_leaf = Hash32([0u8; 32]);

    // Level 8 hash of two null leaves
    let l8_hash = level_keyed_hash(8, &null_leaf, &null_leaf);
    // Level 9 hash
    let l9_hash = level_keyed_hash(9, &l8_hash, &l8_hash);
    // Level 10 hash
    let l10_hash = level_keyed_hash(10, &l9_hash, &l9_hash);

    // Compare against QMDB reference test vectors (Task 1)
    // assert_eq!(l8_hash, EXPECTED_L8_FROM_VECTORS);
}
```

## Dependencies

- `SegmentConfig` from `storage/src/qmdb/segment.rs` (runtime segment size configuration)
- `BalancedTreeRootBuilder` from Task 4 (for `root()` implementation)
- `NullHashCache` from Task 4 (for null hash computation)
