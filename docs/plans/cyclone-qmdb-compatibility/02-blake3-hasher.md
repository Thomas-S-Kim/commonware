# Task 2: Add Blake3 Level-Keyed Hasher

## Current State

- `storage/src/mmr/hasher.rs` has `Standard<H>` using position-based hashing
- Hashing scheme: `H(position || left || right)` with configurable cryptographic hasher
- No support for Blake3 keyed hashing with level as domain separator

## Expected Goal

Add `CycloneHasher` that implements Blake3 keyed hashing with level as domain separator, matching cyclone's exact implementation without modifying existing code.

## Rationale

Cyclone uses a specific hashing scheme:
- Blake3 with level XOR'd into the IV
- Input is always `left || right` (64 bytes for two 32-byte hashes)
- Level provides domain separation instead of position

This cannot be achieved by parameterizing the existing `Standard` hasher - it requires a new implementation.

## Execution Plan

### Step 1: Add Blake3 Dependency

**File**: `storage/Cargo.toml`

```toml
[features]
cyclone-compat = ["blake3"]

[dependencies]
blake3 = { version = "1.5", optional = true }
```

### Step 2: Implement Blake3 Keyed Hash Function

**File**: `storage/src/mmr/cyclone_hasher.rs`

```rust
//! Blake3 level-keyed hasher for cyclone QMDB compatibility.

use alloc::vec::Vec;

/// 32-byte hash output matching cyclone's Hash32 type.
pub type Hash32 = [u8; 32];

/// Zero hash constant.
pub const ZERO_HASH32: Hash32 = [0u8; 32];

/// Blake3 IV converted to bytes (little-endian).
/// Matches cyclone's BLAKE3_IVU8.
static BLAKE3_IV_BYTES: [u8; 32] = [
    0x67, 0xE6, 0x09, 0x6A, // 0x6A09E667
    0x85, 0xAE, 0x67, 0xBB, // 0xBB67AE85
    0x72, 0xF3, 0x6E, 0x3C, // 0x3C6EF372
    0x3A, 0xF5, 0x4F, 0xA5, // 0xA54FF53A
    0x7F, 0x52, 0x0E, 0x51, // 0x510E527F
    0x8C, 0x68, 0x05, 0x9B, // 0x9B05688C
    0xAB, 0xD9, 0x83, 0x1F, // 0x1F83D9AB
    0x19, 0xCD, 0xE0, 0x5B, // 0x5BE0CD19
];

/// Compute Blake3 keyed hash with level XOR'd into IV.
///
/// This matches cyclone's `blake3_keyed_hash64` function exactly.
pub fn blake3_keyed_hash64(level: u8, left: &[u8; 32], right: &[u8; 32]) -> Hash32 {
    // XOR level (as u32 little-endian) into first 4 bytes of IV
    let mut key = BLAKE3_IV_BYTES;
    let level_bytes = (level as u32).to_le_bytes();
    for (i, &byte) in level_bytes.iter().enumerate() {
        key[i] ^= byte;
    }

    // Concatenate left and right into 64-byte input
    let mut input = [0u8; 64];
    input[..32].copy_from_slice(left);
    input[32..].copy_from_slice(right);

    // Compute keyed hash
    blake3::keyed_hash(&key, &input).into()
}

/// Compute merkle node hash at given level.
///
/// This matches cyclone's `merkle_node_hash` function.
pub fn merkle_node_hash(children_level: u8, left: &[u8; 32], right: &[u8; 32]) -> Hash32 {
    blake3_keyed_hash64(children_level, left, right)
}

/// In-place variant that writes directly to target buffer.
pub fn merkle_node_hash_inplace(
    children_level: u8,
    target: &mut [u8; 32],
    left: &[u8; 32],
    right: &[u8; 32],
) {
    *target = merkle_node_hash(children_level, left, right);
}
```

### Step 3: Create CycloneMerkleHasher Trait

**File**: `storage/src/mmr/cyclone_hasher.rs` (continued)

```rust
/// Trait for cyclone-compatible merkle tree hashing.
///
/// Unlike the standard MMR Hasher trait which uses position-based hashing,
/// this trait uses level-based domain separation matching cyclone's approach.
pub trait CycloneMerkleHasher: Send + Sync {
    /// Compute the hash of a leaf node.
    fn leaf_hash(&self, data: &[u8]) -> Hash32;

    /// Compute the hash of an internal node at the given level.
    fn node_hash(&self, level: u8, left: &Hash32, right: &Hash32) -> Hash32;

    /// Fork the hasher for parallel computation.
    fn fork(&self) -> Self;
}

/// Standard implementation using Blake3 keyed hashing.
#[derive(Clone, Default)]
pub struct CycloneHasher;

impl CycloneHasher {
    pub fn new() -> Self {
        Self
    }
}

impl CycloneMerkleHasher for CycloneHasher {
    fn leaf_hash(&self, data: &[u8]) -> Hash32 {
        blake3::hash(data).into()
    }

    fn node_hash(&self, level: u8, left: &Hash32, right: &Hash32) -> Hash32 {
        merkle_node_hash(level, left, right)
    }

    fn fork(&self) -> Self {
        Self
    }
}
```

### Step 4: Export from Module

**File**: `storage/src/mmr/mod.rs`

```rust
#[cfg(feature = "cyclone-compat")]
pub mod cyclone_hasher;

#[cfg(feature = "cyclone-compat")]
pub use cyclone_hasher::{CycloneHasher, CycloneMerkleHasher, Hash32};
```

## Files to Create

- `storage/src/mmr/cyclone_hasher.rs`

## Files to Modify

- `storage/Cargo.toml` (add blake3 dependency and feature flag)
- `storage/src/mmr/mod.rs` (add module export)

## Testing Strategy

### Unit Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_blake3_iv_matches_cyclone() {
        // Verify IV bytes match cyclone's BLAKE3_IVU8
        const CYCLONE_IV: [u32; 8] = [
            0x6A09E667, 0xBB67AE85, 0x3C6EF372, 0xA54FF53A,
            0x510E527F, 0x9B05688C, 0x1F83D9AB, 0x5BE0CD19,
        ];
        for i in 0..8 {
            let bytes = &BLAKE3_IV_BYTES[i * 4..(i + 1) * 4];
            let value = u32::from_le_bytes(bytes.try_into().unwrap());
            assert_eq!(value, CYCLONE_IV[i]);
        }
    }

    #[test]
    fn test_keyed_hash_deterministic() {
        let left = [0xAA; 32];
        let right = [0xBB; 32];
        let h1 = blake3_keyed_hash64(5, &left, &right);
        let h2 = blake3_keyed_hash64(5, &left, &right);
        assert_eq!(h1, h2);
    }

    #[test]
    fn test_level_affects_hash() {
        let left = [0xAA; 32];
        let right = [0xBB; 32];
        let h1 = blake3_keyed_hash64(5, &left, &right);
        let h2 = blake3_keyed_hash64(6, &left, &right);
        assert_ne!(h1, h2);
    }

    #[test]
    fn test_known_vector() {
        // TODO: Add test vector from cyclone
        // let left = hex::decode("...").unwrap();
        // let right = hex::decode("...").unwrap();
        // let expected = hex::decode("...").unwrap();
        // assert_eq!(blake3_keyed_hash64(8, &left, &right), expected);
    }
}
```

### Conformance Tests

Add conformance test to verify hash stability:

```rust
#[cfg(feature = "arbitrary")]
mod conformance {
    // Hash known inputs and compare against stored values
}
```

## Dependencies

- None (can be done in parallel with Task 1)

## Estimated Effort

- 0.5-1 day implementation
- 0.5 day testing and validation against cyclone
