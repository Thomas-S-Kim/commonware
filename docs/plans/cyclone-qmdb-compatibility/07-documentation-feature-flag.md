# Task 7: Documentation and Feature Flag

## Current State

- `StateRootScheme` exists but undocumented
- Feature flag behavior undocumented
- No migration guide

## Goal

- Document `StateRootScheme` abstraction
- Document scheme selection
- Document migration path from reference implementation
- Document default scheme swap timeline

## Execution

### Step 1: Cargo.toml

**File**: `storage/Cargo.toml`

```toml
[features]
default = []
balanced_tree = ["dep:blake3"]

[package.metadata.docs.rs]
all-features = true
```

### Step 2: Module Documentation

**File**: `storage/src/qmdb/scheme.rs` (top)

```rust
//! State root calculation schemes.
//!
//! # Schemes
//!
//! - `BalancedTreeScheme`: Blake3 level-keyed, fixed 2048-entry segments with
//!   hierarchical bitmap. Overlays a virtual balanced binary tree on segment roots.
//! - `MmrScheme`: Position-based hashing, MMR with grafting. Native commonware.
//!
//! # Selection
//!
//! ```rust
//! // Explicit balanced tree
//! let db = Qmdb::<BalancedTreeScheme>::new(config);
//!
//! // Explicit MMR
//! let db = Qmdb::<MmrScheme>::new(config);
//!
//! // Default (balanced_tree when feature enabled, MMR otherwise)
//! let db = Qmdb::<DefaultScheme>::new(config);
//! ```
//!
//! # Migration
//!
//! 1. Enable `balanced_tree` feature
//! 2. Use `Qmdb::<BalancedTreeScheme>` or `Qmdb::<DefaultScheme>`
//! 3. When ready to migrate: switch to `Qmdb::<MmrScheme>`
//! 4. After migration: disable `balanced_tree` feature
```

### Step 3: Example

**File**: `storage/examples/balanced_tree.rs`

```rust
//! cargo run --example balanced_tree --features balanced_tree

use commonware_storage::qmdb::{Qmdb, BalancedTreeScheme};

fn main() {
    let mut db = Qmdb::<BalancedTreeScheme>::new();

    for i in 0..10 {
        let hash = [i as u8; 32];
        db.add(hash);
    }

    let root = db.root();
    println!("root: 0x{}", hex::encode(root));

    db.deactivate(5);
    let new_root = db.root();
    println!("after deactivate: 0x{}", hex::encode(new_root));
}
```

### Step 4: lib.rs Exports

**File**: `storage/src/lib.rs`

```rust
#[cfg(feature = "balanced_tree")]
pub use qmdb::scheme::{BalancedTreeScheme, StateRootScheme, DefaultScheme};
```

## Files to Create

- `storage/examples/balanced_tree.rs`

## Files to Modify

- `storage/Cargo.toml`
- `storage/src/lib.rs`
- `storage/src/qmdb/scheme.rs`

## Testing

```bash
cargo test --doc --features balanced_tree
cargo run --example balanced_tree --features balanced_tree
```

## Dependencies

- Tasks 2-6
