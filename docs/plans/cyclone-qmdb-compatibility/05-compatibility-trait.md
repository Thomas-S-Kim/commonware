# Task 5: Create Cyclone Compatibility Trait

## Current State

- Cyclone-compatible components exist as separate types (CycloneHasher, Twig, CycloneRootBuilder)
- No unified interface tying them together
- No type-safe way to opt into cyclone compatibility mode

## Expected Goal

Define `CycloneCompatible` trait that unifies all cyclone-compatible components and enables type-safe opt-in to cyclone compatibility mode.

## Rationale

A unified trait provides:
1. **Type safety**: Compile-time guarantee that all components are compatible
2. **Ergonomics**: Single point of configuration for cyclone mode
3. **Extensibility**: Future compatibility modes can follow same pattern
4. **Documentation**: Clear API surface for cyclone compatibility

## Execution Plan

### Step 1: Define Core Trait

**File**: `storage/src/qmdb/cyclone_compat.rs`

```rust
//! Unified interface for cyclone QMDB compatibility.

use crate::bitmap::cyclone_twig::{ActiveBits, Twig, LEAF_COUNT_IN_TWIG};
use crate::mmr::cyclone_hasher::{CycloneHasher, CycloneMerkleHasher, Hash32};
use crate::qmdb::cyclone_root::CycloneRootBuilder;

/// Trait for types that can operate in cyclone-compatible mode.
///
/// This trait unifies the hasher, twig structure, and root builder
/// to ensure they work together correctly.
pub trait CycloneCompatible {
    /// The hasher type used for merkle tree computations.
    type Hasher: CycloneMerkleHasher;

    /// Create a new hasher instance.
    fn hasher() -> Self::Hasher;

    /// Create a new empty twig.
    fn new_twig() -> Twig {
        Twig::new()
    }

    /// Create a new root builder.
    fn root_builder() -> CycloneRootBuilder {
        CycloneRootBuilder::new()
    }

    /// Number of entries per twig.
    const ENTRIES_PER_TWIG: u32 = LEAF_COUNT_IN_TWIG;
}

/// Standard cyclone-compatible implementation.
#[derive(Clone, Debug, Default)]
pub struct CycloneMode;

impl CycloneCompatible for CycloneMode {
    type Hasher = CycloneHasher;

    fn hasher() -> Self::Hasher {
        CycloneHasher::new()
    }
}
```

### Step 2: Add Database Wrapper

**File**: `storage/src/qmdb/cyclone_compat.rs` (continued)

```rust
/// A cyclone-compatible authenticated database.
///
/// This wraps the underlying storage and provides cyclone-compatible
/// state root calculation.
pub struct CycloneDb<M: CycloneCompatible = CycloneMode> {
    /// Active twigs indexed by twig ID.
    twigs: std::collections::BTreeMap<u64, TwigState>,

    /// Current serial number (next entry ID).
    next_sn: u64,

    /// Cached state root.
    cached_root: Option<Hash32>,

    /// Marker for the compatibility mode.
    _mode: std::marker::PhantomData<M>,
}

/// State for a single twig.
struct TwigState {
    /// The twig's merkle tree state.
    twig: Twig,

    /// Active bits for this twig.
    active_bits: ActiveBits,

    /// Entry hashes for this twig (merkle tree over entries).
    entry_tree: Vec<Hash32>,

    /// Whether this twig has been modified since last root computation.
    dirty: bool,
}

impl<M: CycloneCompatible> CycloneDb<M> {
    /// Create a new empty database.
    pub fn new() -> Self {
        Self {
            twigs: std::collections::BTreeMap::new(),
            next_sn: 0,
            cached_root: None,
            _mode: std::marker::PhantomData,
        }
    }

    /// Add an entry to the database.
    pub fn add(&mut self, entry_hash: Hash32) -> u64 {
        let sn = self.next_sn;
        self.next_sn += 1;

        let twig_id = sn >> crate::bitmap::cyclone_twig::TWIG_SHIFT;
        let entry_idx = (sn & (M::ENTRIES_PER_TWIG as u64 - 1)) as u32;

        let twig_state = self.twigs.entry(twig_id).or_insert_with(|| {
            TwigState {
                twig: M::new_twig(),
                active_bits: ActiveBits::new(),
                entry_tree: vec![Hash32::default(); M::ENTRIES_PER_TWIG as usize * 2],
                dirty: true,
            }
        });

        // Set active bit
        twig_state.active_bits.set_bit(entry_idx);

        // Store entry hash in tree
        let leaf_idx = M::ENTRIES_PER_TWIG as usize + entry_idx as usize;
        twig_state.entry_tree[leaf_idx] = entry_hash;

        twig_state.dirty = true;
        self.cached_root = None;

        sn
    }

    /// Deactivate an entry.
    pub fn deactivate(&mut self, sn: u64) {
        let twig_id = sn >> crate::bitmap::cyclone_twig::TWIG_SHIFT;
        let entry_idx = (sn & (M::ENTRIES_PER_TWIG as u64 - 1)) as u32;

        if let Some(twig_state) = self.twigs.get_mut(&twig_id) {
            twig_state.active_bits.clear_bit(entry_idx);
            twig_state.dirty = true;
            self.cached_root = None;
        }
    }

    /// Compute and return the current state root.
    pub fn root(&mut self) -> Hash32 {
        if let Some(root) = self.cached_root {
            return root;
        }

        // Sync all dirty twigs
        for twig_state in self.twigs.values_mut() {
            if twig_state.dirty {
                self.sync_twig::<M>(twig_state);
                twig_state.dirty = false;
            }
        }

        // Build root from all twigs
        let mut builder = M::root_builder();
        for (&twig_id, twig_state) in &self.twigs {
            builder.add_twig(twig_id, twig_state.twig.twig_root, false);
        }
        let (_, new_root) = builder.finalize();

        self.cached_root = Some(new_root);
        new_root
    }

    fn sync_twig<Mode: CycloneCompatible>(twig_state: &mut TwigState) {
        // Sync entry tree to compute left_root
        Self::sync_entry_tree::<Mode>(&mut twig_state.entry_tree);
        twig_state.twig.left_root = twig_state.entry_tree[1];

        // Sync active bits tree
        twig_state.twig.sync_all(&twig_state.active_bits);
    }

    fn sync_entry_tree<Mode: CycloneCompatible>(tree: &mut [Hash32]) {
        let n = Mode::ENTRIES_PER_TWIG as usize;
        for level in 0..11 {
            let level_size = n >> level;
            let level_start = level_size;
            for i in (0..level_size).step_by(2) {
                let parent_idx = level_start / 2 + i / 2;
                let left_idx = level_start + i;
                let right_idx = level_start + i + 1;
                tree[parent_idx] = crate::mmr::cyclone_hasher::merkle_node_hash(
                    level as u8,
                    &tree[left_idx],
                    &tree[right_idx],
                );
            }
        }
    }
}

impl<M: CycloneCompatible> Default for CycloneDb<M> {
    fn default() -> Self {
        Self::new()
    }
}
```

### Step 3: Export from Module

**File**: `storage/src/qmdb/mod.rs`

```rust
#[cfg(feature = "cyclone-compat")]
pub mod cyclone_compat;

#[cfg(feature = "cyclone-compat")]
pub use cyclone_compat::{CycloneCompatible, CycloneMode, CycloneDb};
```

## Files to Create

- `storage/src/qmdb/cyclone_compat.rs`

## Files to Modify

- `storage/src/qmdb/mod.rs` (add module export)

## Testing Strategy

### Unit Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_cyclone_db_empty_root() {
        let mut db = CycloneDb::<CycloneMode>::new();
        let root = db.root();
        // Should match cyclone's empty root
    }

    #[test]
    fn test_cyclone_db_single_entry() {
        let mut db = CycloneDb::<CycloneMode>::new();
        let entry_hash = [0xAA; 32];
        let sn = db.add(entry_hash);
        assert_eq!(sn, 0);
        let root = db.root();
        // Should match cyclone's single-entry root
    }

    #[test]
    fn test_cyclone_db_deactivate() {
        let mut db = CycloneDb::<CycloneMode>::new();
        let sn = db.add([0xAA; 32]);
        let root_before = db.root();
        db.deactivate(sn);
        let root_after = db.root();
        assert_ne!(root_before, root_after);
    }
}
```

## Dependencies

- Task 2 (CycloneHasher)
- Task 3 (Twig, ActiveBits)
- Task 4 (CycloneRootBuilder)

## Estimated Effort

- 1 day implementation
- 0.5 day testing
