# Task 03a: Entry Tree Structure

## Goal

Implement the entry tree structure for Milestone 1 (core hashing, no bitmap).

The entry tree is a binary tree of entry hashes with configurable size via `SegmentConfig`:
- Number of entries: `2^shift` (configurable, default 2048 for shift=11)
- Levels 0 to (shift-1): Internal nodes
- Level `shift`: Single root node (entry_root)

This task implements entry tree hashing only. Bitmap integration is in Milestone 2.

## Structure

For default QMDB config (shift=11, 2048 entries):
```
Entry Tree Root (level 11)
└── Binary tree of 2048 entry hashes
    Level 10: 2 nodes
    Level 9:  4 nodes
    Level 8:  8 nodes
    ...
    Level 1:  1024 nodes
    Level 0:  2048 leaves (entry hashes)
```

For custom config (e.g., shift=10, 1024 entries):
```
Entry Tree Root (level 10)
└── Binary tree of 1024 entry hashes
    Level 9:  2 nodes
    Level 8:  4 nodes
    ...
    Level 0:  1024 leaves (entry hashes)
```

## Execution

### Step 1: Use SegmentConfig

**File**: `storage/src/qmdb/entry_tree.rs`

```rust
//! Entry tree for balanced tree scheme.
//!
//! Binary tree of entry hashes with configurable size via SegmentConfig.
//! This is the "left subtree" of a full segment - bitmap integration
//! is handled separately in Milestone 2.

use crate::hasher::level_keyed::Hash32;
use crate::mmr::hasher::Hasher;
use crate::qmdb::segment::SegmentConfig;
```

### Step 2: Entry Tree State

```rust
/// Entry tree state for a single segment.
///
/// Stores entry hashes (count determined by SegmentConfig) and computes the entry tree root.
/// Does NOT include bitmap - that's added in Milestone 2.
#[derive(Clone, Debug)]
pub struct EntryTree {
    /// Segment configuration (determines entries_per_segment)
    config: SegmentConfig,
    /// Entry hashes (runtime-sized based on config)
    pub entries: Box<[Hash32]>,
    /// Cached entry tree root (at level = config.entry_root_level())
    pub entry_root: Hash32,
    /// Whether root needs recomputation
    dirty: bool,
}

impl EntryTree {
    /// Create a new empty entry tree with default QMDB config (2048 entries).
    pub fn new() -> Self {
        Self::with_config(SegmentConfig::default_qmdb())
    }

    /// Create a new empty entry tree with custom segment config.
    pub fn with_config(config: SegmentConfig) -> Self {
        let entries = vec![Hash32::default(); config.entries_per_segment() as usize]
            .into_boxed_slice();
        Self {
            config,
            entries,
            entry_root: Hash32::default(),
            dirty: true,
        }
    }

    /// Get the segment configuration.
    pub fn config(&self) -> SegmentConfig {
        self.config
    }

    /// Set entry hash at index.
    pub fn set_entry(&mut self, index: u32, hash: Hash32) -> Result<(), EntryTreeError> {
        let entries_per_segment = self.config.entries_per_segment();
        if index >= entries_per_segment {
            return Err(EntryTreeError::IndexOutOfBounds(index, entries_per_segment));
        }
        self.entries[index as usize] = hash;
        self.dirty = true;
        Ok(())
    }

    /// Get entry hash at index.
    pub fn get_entry(&self, index: u32) -> Result<&Hash32, EntryTreeError> {
        let entries_per_segment = self.config.entries_per_segment();
        if index >= entries_per_segment {
            return Err(EntryTreeError::IndexOutOfBounds(index, entries_per_segment));
        }
        Ok(&self.entries[index as usize])
    }

    /// Compute entry tree root using provided hasher.
    /// Returns cached value if not dirty.
    pub fn compute_root<H: Hasher<Hash32>>(&mut self, hasher: &mut H) -> Hash32 {
        if !self.dirty {
            return self.entry_root;
        }

        self.entry_root = compute_binary_tree_root(&self.entries, self.config, hasher);
        self.dirty = false;
        self.entry_root
    }

    /// Get cached root (may be stale if dirty).
    pub fn cached_root(&self) -> Hash32 {
        self.entry_root
    }

    /// Check if root needs recomputation.
    pub fn is_dirty(&self) -> bool {
        self.dirty
    }
}

impl Default for EntryTree {
    fn default() -> Self {
        Self::new()
    }
}
```

### Step 3: Tree Root Computation

```rust
/// Compute binary tree root from leaf hashes.
///
/// Builds a complete binary tree with `config.entries_per_segment()` leaves.
/// Uses level-keyed hashing via the Hasher trait.
pub fn compute_binary_tree_root<H: Hasher<Hash32>>(
    leaves: &[Hash32],
    config: SegmentConfig,
    hasher: &mut H,
) -> Hash32 {
    assert_eq!(leaves.len(), config.entries_per_segment() as usize);

    // Level 0: Start with leaves
    let mut current_level: Vec<Hash32> = leaves.to_vec();
    let mut level: u8 = 0;

    // Build tree bottom-up until we reach entry_root_level
    while current_level.len() > 1 {
        let mut next_level = Vec::with_capacity((current_level.len() + 1) / 2);

        for (i, chunk) in current_level.chunks(2).enumerate() {
            // Position encoding for this level
            let pos = ((1u64 << level) - 1) + i as u64;

            let left = &chunk[0];
            let right = if chunk.len() > 1 { &chunk[1] } else { left };

            next_level.push(hasher.node_digest(pos, left, right));
        }

        current_level = next_level;
        level += 1;
    }

    current_level.into_iter().next().unwrap_or_default()
}

/// Build sibling path for entry at given index.
///
/// Returns `config.shift()` sibling hashes for Merkle proof.
/// Path length varies based on segment configuration.
pub fn build_entry_path<H: Hasher<Hash32>>(
    leaves: &[Hash32],
    entry_index: u32,
    config: SegmentConfig,
    hasher: &mut H,
) -> Vec<Hash32> {
    let path_len = config.shift() as usize;
    let mut path = vec![Hash32::default(); path_len];
    let mut current_level: Vec<Hash32> = leaves.to_vec();
    let mut index = entry_index as usize;

    for level in 0..config.shift() {
        // Sibling is at index XOR 1
        let sibling_idx = index ^ 1;
        path[level as usize] = if sibling_idx < current_level.len() {
            current_level[sibling_idx]
        } else {
            Hash32::default() // Null sibling
        };

        // Build next level
        let mut next_level = Vec::with_capacity((current_level.len() + 1) / 2);
        for (i, chunk) in current_level.chunks(2).enumerate() {
            let pos = ((1u64 << level) - 1) + i as u64;
            let left = &chunk[0];
            let right = if chunk.len() > 1 { &chunk[1] } else { left };
            next_level.push(hasher.node_digest(pos, left, right));
        }

        current_level = next_level;
        index /= 2;
    }

    path
}

/// Verify entry hash against entry root using sibling path.
///
/// Path length must match `config.shift()`.
pub fn verify_entry_path<H: Hasher<Hash32>>(
    entry_hash: &Hash32,
    entry_index: u32,
    path: &[Hash32],
    config: SegmentConfig,
    expected_root: &Hash32,
    hasher: &mut H,
) -> bool {
    if path.len() != config.shift() as usize {
        return false; // Path length mismatch
    }

    let mut current = *entry_hash;
    let mut index = entry_index as u64;

    for (level, sibling) in path.iter().enumerate() {
        let pos = ((1u64 << level) - 1) + index / 2;

        current = if index % 2 == 0 {
            hasher.node_digest(pos, &current, sibling)
        } else {
            hasher.node_digest(pos, sibling, &current)
        };

        index /= 2;
    }

    current == *expected_root
}
```

### Step 4: Error Type

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum EntryTreeError {
    #[error("index out of bounds: {0} >= {1}")]
    IndexOutOfBounds(u32, u32),
}
```

### Step 5: Module Export

**File**: `storage/src/qmdb/mod.rs`

```rust
#[cfg(feature = "balanced_tree")]
pub mod entry_tree;
pub mod segment;

#[cfg(feature = "balanced_tree")]
pub use entry_tree::EntryTree;
pub use segment::{SegmentConfig, SegmentConfigError};
```

## Interface Impact

| Interface | Change |
|-----------|--------|
| `EntryTree` | **New struct** (Milestone 1) |
| `compute_binary_tree_root()` | **New function** |
| `build_entry_path()` | **New function** |
| `verify_entry_path()` | **New function** |

No new traits - these are implementation details.

## Testing

```rust
#[test]
fn test_empty_tree_root() {
    let tree = EntryTree::new(); // Uses default QMDB config (2048 entries)
    let mut hasher = LevelKeyed::default();
    let root = tree.compute_root(&mut hasher);
    // Should match null entry tree root from test vectors
}

#[test]
fn test_configurable_segment_size() {
    let config = SegmentConfig::new(10).unwrap(); // 1024 entries
    let tree = EntryTree::with_config(config);
    assert_eq!(tree.entries.len(), 1024);
    assert_eq!(tree.config().shift(), 10);
}

#[test]
fn test_single_entry() {
    let mut tree = EntryTree::new();
    tree.set_entry(0, Hash32([0xAA; 32])).unwrap();

    let mut hasher = LevelKeyed::default();
    let root = tree.compute_root(&mut hasher);
    // Verify against test vector
}

#[test]
fn test_entry_path_verification() {
    let config = SegmentConfig::default_qmdb();
    let mut tree = EntryTree::with_config(config);
    for i in 0..10 {
        tree.set_entry(i, Hash32([i as u8; 32])).unwrap();
    }

    let mut hasher = LevelKeyed::new(config);
    let root = tree.compute_root(&mut hasher);

    // Build and verify path for entry 5
    let path = build_entry_path(&tree.entries, 5, config, &mut hasher);
    assert_eq!(path.len(), config.shift() as usize); // Dynamic path length
    assert!(verify_entry_path(
        &tree.entries[5],
        5,
        &path,
        config,
        &root,
        &mut hasher
    ));
}

#[test]
fn test_different_segment_sizes_produce_different_roots() {
    let data = Hash32([0xAA; 32]);

    // 1024 entries (shift=10)
    let config10 = SegmentConfig::new(10).unwrap();
    let mut tree10 = EntryTree::with_config(config10);
    tree10.set_entry(0, data).unwrap();
    let mut hasher10 = LevelKeyed::new(config10);
    let root10 = tree10.compute_root(&mut hasher10);

    // 2048 entries (shift=11)
    let config11 = SegmentConfig::new(11).unwrap();
    let mut tree11 = EntryTree::with_config(config11);
    tree11.set_entry(0, data).unwrap();
    let mut hasher11 = LevelKeyed::new(config11);
    let root11 = tree11.compute_root(&mut hasher11);

    // Different configs should produce different roots
    assert_ne!(root10, root11);
}

#[test]
fn test_matches_reference_vectors() {
    // Load vectors from 01a-test-vectors-core.md
    let vectors = load_test_vectors("core_hashing.toml");

    for vector in vectors.entry_tree {
        let config = SegmentConfig::default_qmdb(); // Reference uses 2048
        let mut tree = EntryTree::with_config(config);
        for (i, hash) in vector.entries.iter().enumerate() {
            tree.set_entry(i as u32, *hash).unwrap();
        }

        let mut hasher = LevelKeyed::new(config);
        let root = tree.compute_root(&mut hasher);
        assert_eq!(root, vector.entry_root, "Failed: {}", vector.name);
    }
}
```

## Dependencies

- Task 01a (test vectors for validation)
- Task 02 (LevelKeyed hasher)
- `SegmentConfig` from `storage/src/qmdb/segment.rs` (runtime segment size configuration)
