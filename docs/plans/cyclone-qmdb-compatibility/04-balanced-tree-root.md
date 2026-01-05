# Task 4: Balanced Tree Root Calculation

## Current State

QMDB root uses grafted MMR with peak bagging:
```rust
// In qmdb/current/mod.rs
async fn root<E, H, const N: usize>(...) -> Result<H::Digest, Error>
```

The balanced tree scheme needs different finalization - virtual tree overlay instead of peak bagging. This is an **implementation detail** internal to `BalancedTreeScheme`.

## Goal

Add `BalancedTreeRootBuilder` for combining segment roots into a final state root.

This is **not a public trait** - it's used internally by `LevelKeyed::root()` and `BalancedTreeScheme`. The scheme trait itself handles the difference in root finalization.

**CRITICAL**: This component must produce byte-for-byte identical roots to the QMDB reference implementation.

## Algorithm

Stack-based post-order traversal building a virtual balanced binary tree:
1. Each segment root enters as a leaf at level `config.segment_root_level()` (shift+1)
2. Siblings (same level, consecutive indices, left is even) are merged immediately
3. Missing siblings are padded with precomputed null hashes (config-aware)
4. Final root is at the apex

```
State Root (level N)
├── H(segment[0], segment[1]) at level 13
│   ├── segment[0] (level 12)
│   └── segment[1] (level 12)
├── H(segment[2], null) at level 13
│   ├── segment[2] (level 12)
│   └── NULL_12 (level 12)
...
```

## Execution

### Step 1: Null Hash Cache

**File**: `storage/src/qmdb/balanced_tree_root.rs`

```rust
use crate::hasher::level_keyed::{Hash32, LevelKeyed, level_keyed_hash};
use crate::mmr::hasher::Hasher;
use crate::qmdb::segment::SegmentConfig;

pub const MAX_LEVEL: u8 = 64;

/// Cache of precomputed null hashes for each level.
///
/// Null hashes are computed lazily and cached. The cache is config-agnostic
/// since null hash computation only depends on level, not segment size.
///
/// null[0] = H([0; 32], [0; 32]) at level 0
/// null[n] = H(null[n-1], null[n-1]) at level n
#[derive(Clone, Debug)]
pub struct NullHashCache {
    hashes: Vec<Hash32>,
}

impl NullHashCache {
    /// Create a new empty cache.
    pub fn new() -> Self {
        Self { hashes: Vec::new() }
    }

    /// Get null hash for given level, computing and caching if needed.
    pub fn get(&mut self, level: u8) -> Hash32 {
        // Extend cache if needed
        while self.hashes.len() <= level as usize {
            let prev = if self.hashes.is_empty() {
                Hash32::default()
            } else {
                self.hashes[self.hashes.len() - 1]
            };
            let new_level = self.hashes.len() as u8;
            let hash = level_keyed_hash(new_level, &prev, &prev);
            self.hashes.push(hash);
        }
        self.hashes[level as usize]
    }
}

impl Default for NullHashCache {
    fn default() -> Self {
        Self::new()
    }
}

/// Get null hash for the segment root level based on config.
/// This is the null hash used when a segment is empty.
pub fn null_hash_for_config(config: &SegmentConfig) -> Hash32 {
    let mut cache = NullHashCache::new();
    cache.get(config.segment_root_level())
}
```

### Step 2: Stack-Based Builder

```rust
/// Internal node during tree construction.
#[derive(Clone, Debug)]
struct StackNode {
    level: u8,
    index: u64,
    hash: Hash32,
}

/// Builds state root from segment roots using virtual balanced tree.
///
/// Not a public trait - used internally by LevelKeyed::root() and BalancedTreeScheme.
pub struct BalancedTreeRootBuilder<H: Hasher<Hash32>> {
    config: SegmentConfig,
    stack: Vec<StackNode>,
    null_cache: NullHashCache,
    hasher: H,
}

impl<H: Hasher<Hash32>> BalancedTreeRootBuilder<H> {
    pub fn new(config: SegmentConfig, hasher: H) -> Self {
        Self {
            config,
            stack: Vec::new(),
            null_cache: NullHashCache::new(),
            hasher,
        }
    }

    /// Add a segment root. Automatically merges siblings.
    pub fn add_segment(&mut self, segment_id: u64, segment_root: Hash32) {
        self.push(StackNode {
            level: self.config.segment_root_level(),
            index: segment_id,
            hash: segment_root,
        });
    }

    fn push(&mut self, node: StackNode) {
        self.stack.push(node);
        while self.try_merge_top() {}
    }

    fn try_merge_top(&mut self) -> bool {
        if self.stack.len() < 2 { return false; }

        let len = self.stack.len();
        let right = &self.stack[len - 1];
        let left = &self.stack[len - 2];

        // Siblings: same level, consecutive, left is even
        if left.level != right.level { return false; }
        if left.index + 1 != right.index { return false; }
        if left.index % 2 != 0 { return false; }

        let pos = (1u64 << (left.level + 1)) - 1 + left.index / 2;
        let parent_hash = self.hasher.node_digest(pos, &left.hash, &right.hash);

        self.stack.pop();
        self.stack.pop();
        self.stack.push(StackNode {
            level: left.level + 1,
            index: left.index / 2,
            hash: parent_hash,
        });
        true
    }

    /// Finalize: pad with nulls and merge remaining nodes.
    pub fn finalize(mut self) -> Hash32 {
        let segment_root_level = self.config.segment_root_level();

        // Keep padding and merging until single root at index 0
        while self.stack.len() > 1 || (self.stack.len() == 1 && self.stack[0].index != 0) {
            self.pad_with_null();
            while self.try_merge_top() {}
        }

        if self.stack.is_empty() {
            return self.null_cache.get(segment_root_level);
        }
        self.stack.pop().unwrap().hash
    }

    fn pad_with_null(&mut self) {
        if self.stack.is_empty() { return; }
        let top = self.stack.last().unwrap();
        if top.index % 2 == 0 {
            let level = top.level;
            let null_hash = self.null_cache.get(level);
            self.stack.push(StackNode {
                level,
                index: top.index + 1,
                hash: null_hash,
            });
        }
    }
}
```

### Step 3: Integration with Hasher and Scheme

The builder is used internally by `LevelKeyed::root()` and `BalancedTreeScheme`:

```rust
// In LevelKeyed::root() implementation (from 02-level-keyed-hasher.md)
fn root<'a>(&mut self, size: Position, peaks: impl Iterator<Item = &'a Hash32>) -> Hash32 {
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

// In scheme/balanced_tree.rs
impl BalancedTreeScheme {
    fn compute_root(&self, chunks: &[SegmentState], hasher: &mut LevelKeyed) -> Hash32 {
        let mut builder = BalancedTreeRootBuilder::new(self.config, hasher.fork());
        for (id, chunk) in chunks.iter().enumerate() {
            builder.add_segment(id as u64, chunk.segment.segment_root);
        }
        builder.finalize()
    }
}
```

### Step 4: Export

**File**: `storage/src/qmdb/mod.rs`

```rust
#[cfg(feature = "balanced_tree")]
mod balanced_tree_root;
#[cfg(feature = "balanced_tree")]
pub(crate) use balanced_tree_root::BalancedTreeRootBuilder;
```

## Interface Impact

| Interface | Change |
|-----------|--------|
| Public traits | **No change** |
| `BalancedTreeRootBuilder` | **New struct** (internal to scheme) |
| `null_hash()` | **New function** (internal) |

No new public traits - this is implementation detail of `BalancedTreeScheme`.

## Files to Create

- `storage/src/qmdb/balanced_tree_root.rs`

## Files to Modify

- `storage/src/qmdb/mod.rs`

## Testing

```rust
#[test]
fn test_empty_builder() {
    let config = SegmentConfig::default_qmdb();
    let builder = BalancedTreeRootBuilder::new(config, LevelKeyed::new(config));
    let root = builder.finalize();
    assert_eq!(root, null_hash_for_config(&config));
}

#[test]
fn test_single_segment() {
    let config = SegmentConfig::default_qmdb();
    let mut builder = BalancedTreeRootBuilder::new(config, LevelKeyed::new(config));
    builder.add_segment(0, Hash32([0xAA; 32]));
    let root = builder.finalize();
    // Single segment at index 0 gets paired with null sibling
    // Verify against test vector
}

#[test]
fn test_two_segments_merge() {
    let config = SegmentConfig::default_qmdb();
    let mut builder = BalancedTreeRootBuilder::new(config, LevelKeyed::new(config));
    builder.add_segment(0, Hash32([0xAA; 32]));
    builder.add_segment(1, Hash32([0xBB; 32]));
    let root = builder.finalize();
    // Should be H([0xAA; 32], [0xBB; 32]) at level 13 (segment_root_level + 1)
}

#[test]
fn test_three_segments_with_padding() {
    let config = SegmentConfig::default_qmdb();
    let mut builder = BalancedTreeRootBuilder::new(config, LevelKeyed::new(config));
    builder.add_segment(0, Hash32([0xAA; 32]));
    builder.add_segment(1, Hash32([0xBB; 32]));
    builder.add_segment(2, Hash32([0xCC; 32]));
    let root = builder.finalize();
    // Segment 2 gets paired with null, then merged up
}

#[test]
fn test_different_segment_configs() {
    // Test that different configs produce different roots for same input
    let seg10 = Hash32([0xAA; 32]);
    let seg11 = Hash32([0xBB; 32]);

    // Config with shift=10 (1024 entries, segment roots at level 11)
    let config10 = SegmentConfig::new(10).unwrap();
    let mut builder10 = BalancedTreeRootBuilder::new(config10, LevelKeyed::new(config10));
    builder10.add_segment(0, seg10);
    builder10.add_segment(1, seg11);
    let root10 = builder10.finalize();

    // Config with shift=11 (2048 entries, segment roots at level 12)
    let config11 = SegmentConfig::new(11).unwrap();
    let mut builder11 = BalancedTreeRootBuilder::new(config11, LevelKeyed::new(config11));
    builder11.add_segment(0, seg10);
    builder11.add_segment(1, seg11);
    let root11 = builder11.finalize();

    // Different configs produce different roots (different starting levels)
    assert_ne!(root10, root11);
}

#[test]
fn test_null_hash_cache() {
    let mut cache = NullHashCache::new();

    // Level 0: H(null, null) at level 0
    let l0 = cache.get(0);
    assert_ne!(l0, Hash32::default()); // Not just zeros

    // Level 1: H(l0, l0) at level 1
    let l1 = cache.get(1);
    assert_ne!(l1, l0); // Different from level 0

    // Cache should return same values on subsequent calls
    assert_eq!(cache.get(0), l0);
    assert_eq!(cache.get(1), l1);
}
```

## Dependencies

- Task 2 (LevelKeyed implementing Hasher<D>)
- Task 3 (Segment structure)
- `SegmentConfig` from `storage/src/qmdb/segment.rs` (runtime segment size configuration)
