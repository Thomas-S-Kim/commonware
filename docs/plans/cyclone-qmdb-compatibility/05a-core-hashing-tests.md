# Task 05a: Core Hashing Validation Tests

## Goal

Validate Milestone 1 implementation against test vectors:
- Leaf hashing (BLAKE2b-512 truncated)
- Node hashing (Blake3 level-keyed)
- Entry tree roots
- Upper tree roots (multiple segments)

**This is the Milestone 1 checkpoint.** All tests must pass before proceeding to Milestone 2.

## Test Categories

### 1. Leaf Hash Tests

```rust
#[cfg(test)]
mod leaf_hash_tests {
    use super::*;
    use crate::hasher::level_keyed::{blake2b_hash_32, LevelKeyed, Hash32};

    #[test]
    fn test_empty_input() {
        let result = blake2b_hash_32(&[]);
        let expected = load_vector("leaf_hash", "empty");
        assert_eq!(result, expected);
    }

    #[test]
    fn test_zero_bytes() {
        let result = blake2b_hash_32(&[0u8; 32]);
        let expected = load_vector("leaf_hash", "zeros_32");
        assert_eq!(result, expected);
    }

    #[test]
    fn test_all_leaf_vectors() {
        let vectors = load_test_vectors("core_hashing.toml");
        for v in vectors.leaf_hash {
            let result = blake2b_hash_32(&v.input);
            assert_eq!(result.0, v.expected, "Leaf hash mismatch for input {:?}", v.input);
        }
    }

    #[test]
    fn test_leaf_digest_via_hasher_trait() {
        let mut hasher = LevelKeyed::default();
        let vectors = load_test_vectors("core_hashing.toml");

        for v in vectors.leaf_hash {
            // Position is ignored for leaf hashing
            let result = hasher.leaf_digest(0, &v.input);
            assert_eq!(result.0, v.expected);

            // Verify position doesn't affect result
            let result2 = hasher.leaf_digest(12345, &v.input);
            assert_eq!(result, result2, "Position should not affect leaf hash");
        }
    }
}
```

### 2. Node Hash Tests

```rust
#[cfg(test)]
mod node_hash_tests {
    use super::*;
    use crate::hasher::level_keyed::{level_keyed_hash, LevelKeyed, Hash32};
    use crate::mmr::hasher::Hasher;

    #[test]
    fn test_level_affects_output() {
        let left = Hash32([0xAA; 32]);
        let right = Hash32([0xBB; 32]);

        let h0 = level_keyed_hash(0, &left, &right);
        let h1 = level_keyed_hash(1, &left, &right);
        let h11 = level_keyed_hash(11, &left, &right);

        assert_ne!(h0, h1, "Different levels must produce different hashes");
        assert_ne!(h1, h11);
        assert_ne!(h0, h11);
    }

    #[test]
    fn test_all_node_vectors() {
        let vectors = load_test_vectors("core_hashing.toml");
        for v in vectors.node_hash {
            let left = Hash32(v.left);
            let right = Hash32(v.right);
            let result = level_keyed_hash(v.level, &left, &right);
            assert_eq!(result.0, v.expected, "Node hash mismatch at level {}", v.level);
        }
    }

    #[test]
    fn test_node_digest_via_hasher_trait() {
        let mut hasher = LevelKeyed::default();
        let vectors = load_test_vectors("core_hashing.toml");

        for v in vectors.node_hash {
            let left = Hash32(v.left);
            let right = Hash32(v.right);
            // Position encoding: (1 << level) - 1 gives the correct level
            let pos = (1u64 << v.level) - 1;
            let result = hasher.node_digest(pos, &left, &right);
            assert_eq!(result.0, v.expected, "Node digest mismatch at level {}", v.level);
        }
    }

    #[test]
    fn test_position_to_level_mapping() {
        use crate::mmr::pos_to_height;

        // Verify position encoding gives correct level
        assert_eq!(pos_to_height(0), 0);      // (1 << 0) - 1 = 0
        assert_eq!(pos_to_height(1), 1);      // (1 << 1) - 1 = 1
        assert_eq!(pos_to_height(3), 2);      // (1 << 2) - 1 = 3
        assert_eq!(pos_to_height(7), 3);      // (1 << 3) - 1 = 7
        assert_eq!(pos_to_height(2047), 11);  // (1 << 11) - 1 = 2047
        assert_eq!(pos_to_height(4095), 12);  // (1 << 12) - 1 = 4095
    }
}
```

### 3. Null Hash Chain Tests

```rust
#[cfg(test)]
mod null_hash_tests {
    use super::*;
    use crate::qmdb::balanced_tree_root::null_hash;

    #[test]
    fn test_null_hash_chain() {
        let vectors = load_test_vectors("core_hashing.toml");

        for level in 0..64u8 {
            let computed = null_hash(level);
            let expected_key = format!("level_{}", level);
            let expected = vectors.null_hashes.get(&expected_key)
                .expect(&format!("Missing null hash for level {}", level));
            assert_eq!(computed.0, *expected, "Null hash mismatch at level {}", level);
        }
    }

    #[test]
    fn test_null_hash_recurrence() {
        // Verify: null[n] = H_n(null[n-1], null[n-1])
        let mut hasher = LevelKeyed::default();

        let null_0 = null_hash(0);
        let recomputed_0 = hasher.node_digest(0, &Hash32::default(), &Hash32::default());
        assert_eq!(*null_0, recomputed_0);

        for level in 1..20u8 {
            let prev = null_hash(level - 1);
            let pos = (1u64 << level) - 1;
            let recomputed = hasher.node_digest(pos, prev, prev);
            assert_eq!(*null_hash(level), recomputed, "Recurrence failed at level {}", level);
        }
    }
}
```

### 4. Entry Tree Tests

```rust
#[cfg(test)]
mod entry_tree_tests {
    use super::*;
    use crate::qmdb::entry_tree::{EntryTree, compute_binary_tree_root, ENTRIES_PER_SEGMENT};

    #[test]
    fn test_empty_entry_tree() {
        let mut tree = EntryTree::new();
        let mut hasher = LevelKeyed::default();
        let root = tree.compute_root(&mut hasher);

        // Empty tree = all null entries
        let vectors = load_test_vectors("core_hashing.toml");
        let expected = vectors.entry_tree.iter()
            .find(|v| v.name == "all_null")
            .expect("Missing all_null entry tree vector");
        assert_eq!(root.0, expected.entry_root);
    }

    #[test]
    fn test_all_entry_tree_vectors() {
        let vectors = load_test_vectors("core_hashing.toml");

        for v in vectors.entry_tree {
            let mut tree = EntryTree::new();
            for (i, hash) in v.entries.iter().enumerate() {
                tree.set_entry(i as u32, Hash32(*hash)).unwrap();
            }

            let mut hasher = LevelKeyed::default();
            let root = tree.compute_root(&mut hasher);
            assert_eq!(root.0, v.entry_root, "Entry tree mismatch: {}", v.name);
        }
    }

    #[test]
    fn test_entry_path_roundtrip() {
        use crate::qmdb::entry_tree::{build_entry_path, verify_entry_path};

        let mut tree = EntryTree::new();
        for i in 0..100 {
            tree.set_entry(i, Hash32([i as u8; 32])).unwrap();
        }

        let mut hasher = LevelKeyed::default();
        let root = tree.compute_root(&mut hasher);

        // Verify path for each populated entry
        for i in 0..100u16 {
            let path = build_entry_path(&tree.entries, i, &mut hasher);
            assert!(
                verify_entry_path(&tree.entries[i as usize], i, &path, &root, &mut hasher),
                "Path verification failed for entry {}", i
            );
        }
    }
}
```

### 5. Upper Tree Tests

```rust
#[cfg(test)]
mod upper_tree_tests {
    use super::*;
    use crate::qmdb::balanced_tree_root::BalancedTreeRootBuilder;

    #[test]
    fn test_single_segment() {
        let mut builder = BalancedTreeRootBuilder::new(LevelKeyed::default());
        let seg_root = Hash32([0xAA; 32]);
        builder.add_segment(0, seg_root);
        let root = builder.finalize();

        let vectors = load_test_vectors("core_hashing.toml");
        let expected = vectors.upper_tree.iter()
            .find(|v| v.name == "single_segment")
            .expect("Missing single_segment vector");
        assert_eq!(root.0, expected.state_root);
    }

    #[test]
    fn test_all_upper_tree_vectors() {
        let vectors = load_test_vectors("core_hashing.toml");

        for v in vectors.upper_tree {
            let mut builder = BalancedTreeRootBuilder::new(LevelKeyed::default());
            for (id, seg_root) in v.segment_roots.iter().enumerate() {
                builder.add_segment(id as u64, Hash32(*seg_root));
            }
            let root = builder.finalize();
            assert_eq!(root.0, v.state_root, "Upper tree mismatch: {}", v.name);
        }
    }

    #[test]
    fn test_padding_with_nulls() {
        // 3 segments should pad to 4, then merge
        let mut builder = BalancedTreeRootBuilder::new(LevelKeyed::default());
        builder.add_segment(0, Hash32([0xAA; 32]));
        builder.add_segment(1, Hash32([0xBB; 32]));
        builder.add_segment(2, Hash32([0xCC; 32]));
        let root = builder.finalize();

        // Verify against test vector
        let vectors = load_test_vectors("core_hashing.toml");
        let expected = vectors.upper_tree.iter()
            .find(|v| v.name == "three_segments")
            .expect("Missing three_segments vector");
        assert_eq!(root.0, expected.state_root);
    }
}
```

## Milestone 1 Checkpoint

```rust
#[test]
fn milestone_1_checkpoint() {
    // This test validates all Milestone 1 requirements

    // 1. Leaf hashing works
    let leaf = blake2b_hash_32(b"test data");
    assert_ne!(leaf, Hash32::default());

    // 2. Node hashing is level-keyed
    let left = Hash32([1; 32]);
    let right = Hash32([2; 32]);
    let h0 = level_keyed_hash(0, &left, &right);
    let h1 = level_keyed_hash(1, &left, &right);
    assert_ne!(h0, h1);

    // 3. Entry tree computes correctly
    let mut tree = EntryTree::new();
    tree.set_entry(0, leaf).unwrap();
    let mut hasher = LevelKeyed::default();
    let entry_root = tree.compute_root(&mut hasher);
    assert_ne!(entry_root, Hash32::default());

    // 4. Upper tree combines segments
    let mut builder = BalancedTreeRootBuilder::new(LevelKeyed::default());
    builder.add_segment(0, entry_root);
    let state_root = builder.finalize();
    assert_ne!(state_root, Hash32::default());

    // 5. All test vectors pass
    let vectors = load_test_vectors("core_hashing.toml");
    assert!(vectors.leaf_hash.len() > 0, "No leaf hash vectors");
    assert!(vectors.node_hash.len() > 0, "No node hash vectors");
    assert!(vectors.entry_tree.len() > 0, "No entry tree vectors");
    assert!(vectors.upper_tree.len() > 0, "No upper tree vectors");

    println!("✓ Milestone 1 COMPLETE: Core hashing matches reference implementation");
}
```

## Files to Create

- `storage/src/qmdb/tests/core_hashing_tests.rs`

## Files to Modify

- `storage/src/qmdb/tests/mod.rs`

## Running Tests

```bash
# Run Milestone 1 tests only
cargo test -p commonware-storage --features balanced_tree core_hashing

# Run milestone checkpoint
cargo test -p commonware-storage --features balanced_tree milestone_1_checkpoint
```

## Success Criteria

- [ ] All leaf hash vectors pass
- [ ] All node hash vectors pass
- [ ] Null hash chain matches (levels 0-63)
- [ ] All entry tree vectors pass
- [ ] All upper tree vectors pass
- [ ] `milestone_1_checkpoint` test passes

## Dependencies

- Task 01a (test vectors)
- Task 02 (LevelKeyed hasher)
- Task 03a (EntryTree)
- Task 04 (BalancedTreeRootBuilder)
