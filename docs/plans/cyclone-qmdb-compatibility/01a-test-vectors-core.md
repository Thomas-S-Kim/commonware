# Task 01a: Core Hashing Test Vectors

## Goal

Generate test vectors for **core hashing behavior only** (no bitmap):
- Leaf hashing (BLAKE2b-512 truncated to 32 bytes)
- Node hashing (Blake3 level-keyed)
- Entry tree root (2048 entries, levels 0-11)
- Upper tree root (multiple segments, levels 12+)

These vectors validate Milestone 1 before adding bitmap complexity.

## Test Vectors to Generate

### 1. Leaf Hash Vectors

```toml
[[leaf_hash]]
input = "0x..." # Raw bytes
expected = "0x..." # BLAKE2b-512 truncated to 32 bytes

[[leaf_hash]]
input = "" # Empty input
expected = "0x..."

[[leaf_hash]]
input = "0x00000000..." # 32 zero bytes
expected = "0x..."
```

### 2. Node Hash Vectors (Level-Keyed)

```toml
[[node_hash]]
level = 0
left = "0x..." # 32 bytes
right = "0x..." # 32 bytes
expected = "0x..." # Blake3 keyed with level XOR'd into IV

[[node_hash]]
level = 5
left = "0xaa..."
right = "0xbb..."
expected = "0x..."

[[node_hash]]
level = 11 # Segment root level
left = "0x..."
right = "0x..."
expected = "0x..."
```

### 3. Null Hash Chain

```toml
[null_hashes]
# Precomputed null hashes for each level
# null[0] = node_hash(0, [0;32], [0;32])
# null[n] = node_hash(n, null[n-1], null[n-1])
level_0 = "0x..."
level_1 = "0x..."
level_2 = "0x..."
# ... up to level 63
level_11 = "0x..." # Entry tree null root
level_12 = "0x..." # Segment null root
level_13 = "0x..." # First upper tree level
```

### 4. Entry Tree Vectors (No Bitmap)

```toml
[[entry_tree]]
name = "single_entry"
entries = ["0x..."] # Single entry hash
entry_root = "0x..." # Root of 2048-entry tree (2047 nulls)

[[entry_tree]]
name = "two_entries"
entries = ["0x...", "0x..."]
entry_root = "0x..."

[[entry_tree]]
name = "full_segment"
entries = [...] # 2048 entry hashes
entry_root = "0x..."
```

### 5. Upper Tree Vectors

```toml
[[upper_tree]]
name = "single_segment"
segment_roots = ["0x..."] # One segment root at level 12
state_root = "0x..." # Padded with null to form root

[[upper_tree]]
name = "two_segments"
segment_roots = ["0x...", "0x..."]
state_root = "0x..." # H(seg0, seg1) at level 13

[[upper_tree]]
name = "three_segments"
segment_roots = ["0x...", "0x...", "0x..."]
state_root = "0x..." # Third padded with null
```

## Execution

### Step 1: Create Vector Generator

**File**: `scripts/generate_core_vectors.rs` (or add to QMDB test harness)

```rust
use qmdb_common::utils::hasher::{blake2b_hash, blake3_keyed_hash64};
use qmdb_common::merkletree::hash::merkle_node_hash_inplace;

fn generate_leaf_vectors() -> Vec<LeafVector> {
    vec![
        LeafVector {
            input: vec![],
            expected: blake2b_hash(&[&[]]),
        },
        LeafVector {
            input: vec![0u8; 32],
            expected: blake2b_hash(&[&[0u8; 32]]),
        },
        // ... more cases
    ]
}

fn generate_node_vectors() -> Vec<NodeVector> {
    let mut vectors = Vec::new();

    for level in [0, 5, 8, 11, 12, 13] {
        let left = [0xAA; 32];
        let right = [0xBB; 32];
        let mut result = [0u8; 32];
        merkle_node_hash_inplace(level, &mut result, &left, &right);

        vectors.push(NodeVector {
            level,
            left,
            right,
            expected: result,
        });
    }
    vectors
}

fn generate_null_chain() -> [Hash32; 64] {
    let mut nulls = [[0u8; 32]; 64];
    for level in 0..64 {
        let prev = if level == 0 { [0u8; 32] } else { nulls[level - 1] };
        merkle_node_hash_inplace(level as u8, &mut nulls[level], &prev, &prev);
    }
    nulls
}
```

### Step 2: Generate and Save Vectors

```bash
cd /path/to/qmdb
cargo run --bin generate_vectors -- --output vectors/core_hashing.toml
```

### Step 3: Validate Format

```rust
#[test]
fn test_vectors_parseable() {
    let vectors: CoreVectors = toml::from_str(include_str!("vectors/core_hashing.toml")).unwrap();
    assert!(!vectors.leaf_hash.is_empty());
    assert!(!vectors.node_hash.is_empty());
    assert!(!vectors.null_hashes.is_empty());
}
```

## Output Location

```
storage/src/qmdb/tests/fixtures/
├── core_hashing.toml      # This task's output
└── bitmap_hashing.toml    # Task 01b output
```

## Success Criteria

- [ ] Leaf hash vectors cover: empty, zeros, random data
- [ ] Node hash vectors cover: levels 0, 5, 8, 11, 12, 13
- [ ] Null hash chain complete (levels 0-63)
- [ ] Entry tree vectors: 1, 2, 2048 entries
- [ ] Upper tree vectors: 1, 2, 3 segments
- [ ] All vectors generated from QMDB reference implementation

## Dependencies

None (first task in Milestone 1).
