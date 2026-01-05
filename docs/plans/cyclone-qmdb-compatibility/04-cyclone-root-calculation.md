# Task 4: Add Cyclone-Compatible Root Calculation

## Current State

- Root calculation in `qmdb/current/mod.rs` uses grafted MMR approach
- Uses peak iteration and MMR-specific root computation
- No support for cyclone's stack-based post-order traversal

## Expected Goal

Add `CycloneRootBuilder` that computes state roots matching cyclone's algorithm using stack-based post-order tree traversal.

## Rationale

Cyclone uses a fundamentally different approach to root calculation:
- Stack-based post-order traversal vs. MMR peak iteration
- Incremental building with old/new state tracking
- Level-based null hash substitution

A separate implementation is cleaner than trying to adapt the grafting approach.

## Execution Plan

### Step 1: Implement Stack Node Structure

**File**: `storage/src/qmdb/cyclone_root.rs`

```rust
//! Cyclone-compatible root calculation using stack-based traversal.

use crate::bitmap::cyclone_twig::{Twig, LEAF_COUNT_IN_TWIG, TWIG_SHIFT};
use crate::bitmap::cyclone_null::get_null_hash_by_level;
use crate::mmr::cyclone_hasher::{Hash32, merkle_node_hash};

/// First level above twig roots.
pub const FIRST_LEVEL_ABOVE_TWIG: u8 = 13;

/// Twig root level.
pub const TWIG_ROOT_LEVEL: u8 = 12;

/// Maximum tree level.
pub const MAX_TREE_LEVEL: u8 = 64;

/// A node on the root builder stack.
#[derive(Clone, Debug)]
struct StackNode {
    /// Level in the tree (12 for twig roots, 13+ for upper tree).
    level: u8,

    /// Position within the level (0-indexed from left).
    nth: u64,

    /// Hash for the old state (None if null).
    old_hash: Option<Hash32>,

    /// Hash for the new state (None if same as old).
    new_hash: Option<Hash32>,

    /// Whether this node was null in old state.
    is_null_for_old: bool,

    /// Whether new state is same as old.
    same_with_old: bool,
}

impl StackNode {
    fn new_twig(nth: u64, twig_root: Hash32, is_new: bool) -> Self {
        Self {
            level: TWIG_ROOT_LEVEL,
            nth,
            old_hash: if is_new { None } else { Some(twig_root) },
            new_hash: Some(twig_root),
            is_null_for_old: is_new,
            same_with_old: !is_new,
        }
    }

    fn get_old_hash(&self) -> &Hash32 {
        self.old_hash.as_ref().unwrap_or_else(|| {
            get_null_hash_by_level(self.level)
        })
    }

    fn get_new_hash(&self) -> &Hash32 {
        if self.same_with_old {
            self.get_old_hash()
        } else {
            self.new_hash.as_ref().unwrap()
        }
    }
}
```

### Step 2: Implement Root Builder

**File**: `storage/src/qmdb/cyclone_root.rs` (continued)

```rust
/// Stack-based root builder matching cyclone's algorithm.
pub struct CycloneRootBuilder {
    /// The computation stack.
    stack: Vec<StackNode>,

    /// Final old state root.
    old_root: Option<Hash32>,

    /// Final new state root.
    new_root: Option<Hash32>,
}

impl CycloneRootBuilder {
    pub fn new() -> Self {
        Self {
            stack: Vec::new(),
            old_root: None,
            new_root: None,
        }
    }

    /// Add a twig root to the builder.
    pub fn add_twig(&mut self, twig_id: u64, twig_root: Hash32, is_new: bool) {
        let node = StackNode::new_twig(twig_id, twig_root, is_new);
        self.push(node);
    }

    /// Push a node and merge siblings if possible.
    fn push(&mut self, node: StackNode) {
        self.stack.push(node);
        while self.try_merge_top() {}
    }

    /// Try to merge the top two nodes if they are siblings.
    fn try_merge_top(&mut self) -> bool {
        if self.stack.len() < 2 {
            return false;
        }

        let len = self.stack.len();
        let right = &self.stack[len - 1];
        let left = &self.stack[len - 2];

        // Check if they are siblings (same level, consecutive positions)
        if left.level != right.level {
            return false;
        }
        if left.nth + 1 != right.nth {
            return false;
        }
        if left.nth % 2 != 0 {
            return false;
        }

        // Compute parent hashes
        let parent_level = left.level + 1;
        let parent_nth = left.nth / 2;

        let old_hash = if left.is_null_for_old && right.is_null_for_old {
            None
        } else {
            Some(merkle_node_hash(
                left.level,
                left.get_old_hash(),
                right.get_old_hash(),
            ))
        };

        let (new_hash, same_with_old) = if left.same_with_old && right.same_with_old {
            (None, true)
        } else {
            (
                Some(merkle_node_hash(
                    left.level,
                    left.get_new_hash(),
                    right.get_new_hash(),
                )),
                false,
            )
        };

        // Pop children and push parent
        self.stack.pop();
        self.stack.pop();
        self.stack.push(StackNode {
            level: parent_level,
            nth: parent_nth,
            old_hash,
            new_hash,
            is_null_for_old: left.is_null_for_old && right.is_null_for_old,
            same_with_old,
        });

        true
    }

    /// Finalize and return (old_root, new_root).
    pub fn finalize(mut self) -> (Hash32, Hash32) {
        // Handle remaining unpaired nodes by pairing with null siblings
        while self.stack.len() > 1 ||
              (self.stack.len() == 1 && self.stack[0].nth != 0) {
            self.pad_with_null();
            while self.try_merge_top() {}
        }

        if self.stack.is_empty() {
            let null = *get_null_hash_by_level(TWIG_ROOT_LEVEL);
            return (null, null);
        }

        let root = self.stack.pop().unwrap();
        (
            *root.get_old_hash(),
            *root.get_new_hash(),
        )
    }

    /// Pad stack with null sibling for incomplete trees.
    fn pad_with_null(&mut self) {
        if self.stack.is_empty() {
            return;
        }

        let top = self.stack.last().unwrap();
        if top.nth % 2 == 0 {
            // Need right sibling
            let null_node = StackNode {
                level: top.level,
                nth: top.nth + 1,
                old_hash: None,
                new_hash: None,
                is_null_for_old: true,
                same_with_old: true,
            };
            self.stack.push(null_node);
        }
        // If nth is odd, parent will handle it
    }
}

impl Default for CycloneRootBuilder {
    fn default() -> Self {
        Self::new()
    }
}
```

### Step 3: Add Helper Functions

**File**: `storage/src/qmdb/cyclone_root.rs` (continued)

```rust
/// Calculate max level needed for given twig count.
pub fn calc_max_level(youngest_twig_id: u64) -> u8 {
    if youngest_twig_id == 0 {
        return TWIG_ROOT_LEVEL;
    }
    (FIRST_LEVEL_ABOVE_TWIG as u64 + 63 - youngest_twig_id.leading_zeros() as u64) as u8
}

/// Calculate max level from serial number.
pub fn calc_max_level_from_sn(sn: u64) -> u8 {
    calc_max_level(sn >> TWIG_SHIFT)
}

/// Compute root from a sequence of twigs.
pub fn compute_root<'a>(
    twigs: impl Iterator<Item = (u64, &'a Twig, bool)>,
) -> (Hash32, Hash32) {
    let mut builder = CycloneRootBuilder::new();
    for (twig_id, twig, is_new) in twigs {
        builder.add_twig(twig_id, twig.twig_root, is_new);
    }
    builder.finalize()
}
```

### Step 4: Export from Module

**File**: `storage/src/qmdb/mod.rs`

```rust
#[cfg(feature = "cyclone-compat")]
pub mod cyclone_root;

#[cfg(feature = "cyclone-compat")]
pub use cyclone_root::{CycloneRootBuilder, compute_root};
```

## Files to Create

- `storage/src/qmdb/cyclone_root.rs`

## Files to Modify

- `storage/src/qmdb/mod.rs` (add module export)

## Testing Strategy

### Unit Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_single_twig_root() {
        let mut builder = CycloneRootBuilder::new();
        let twig_root = [0xAA; 32];
        builder.add_twig(0, twig_root, false);
        let (old, new) = builder.finalize();
        // Single twig at position 0 needs null siblings up to root
        assert_ne!(old, twig_root); // Should include null padding
    }

    #[test]
    fn test_two_twigs_merge() {
        let mut builder = CycloneRootBuilder::new();
        builder.add_twig(0, [0xAA; 32], false);
        builder.add_twig(1, [0xBB; 32], false);
        let (old, new) = builder.finalize();
        assert_eq!(old, new); // Both not new, should match
    }

    #[test]
    fn test_matches_cyclone_vectors() {
        // TODO: Use test vectors from Task 1
    }
}
```

## Dependencies

- Task 2 (Blake3 hasher)
- Task 3 (Twig structure and null hashes)

## Estimated Effort

- 1-2 days implementation
- 0.5 day testing against cyclone vectors
