# Task 7: Documentation and Feature Flag

## Current State

- New cyclone-compat code exists but is not documented
- Feature flag is used internally but not properly exposed
- No usage examples or migration guide

## Expected Goal

Complete documentation for cyclone compatibility mode including:
- Module-level documentation
- API documentation for all public types
- Usage examples
- Feature flag documentation in Cargo.toml

## Rationale

Good documentation:
1. Enables adoption without reading source code
2. Serves as specification for expected behavior
3. Reduces support burden
4. Demonstrates correct usage patterns

## Execution Plan

### Step 1: Update Cargo.toml

**File**: `storage/Cargo.toml`

```toml
[features]
default = []

# Enable cyclone QMDB compatibility mode.
# This adds support for computing state roots that match cyclone's QMDB implementation.
# Adds dependency on blake3.
cyclone-compat = ["dep:blake3"]

[dependencies]
blake3 = { version = "1.5", optional = true }

[package.metadata.docs.rs]
all-features = true
rustdoc-args = ["--cfg", "docsrs"]
```

### Step 2: Add Module Documentation

**File**: `storage/src/qmdb/cyclone_compat.rs` (top of file)

```rust
//! Cyclone QMDB compatibility mode.
//!
//! This module provides types and traits for computing state roots that are
//! byte-for-byte identical to [cyclone's QMDB implementation](https://github.com/LayerZero-Research/cyclone).
//!
//! # Overview
//!
//! Cyclone QMDB uses a different internal structure than commonware's native QMDB:
//!
//! | Aspect | Commonware Native | Cyclone Compatible |
//! |--------|-------------------|-------------------|
//! | Hash function | Position-based | Blake3 level-keyed |
//! | Tree structure | MMR with grafting | Fixed twig structure |
//! | Entries per chunk | Configurable | Fixed 2048 |
//!
//! # Usage
//!
//! Enable the `cyclone-compat` feature in your `Cargo.toml`:
//!
//! ```toml
//! [dependencies]
//! commonware-storage = { version = "...", features = ["cyclone-compat"] }
//! ```
//!
//! Then use the [`CycloneDb`] type:
//!
//! ```rust,ignore
//! use commonware_storage::qmdb::{CycloneDb, CycloneMode};
//!
//! let mut db = CycloneDb::<CycloneMode>::new();
//!
//! // Add entries
//! let entry_hash = [0xAA; 32];
//! let serial_number = db.add(entry_hash);
//!
//! // Compute state root (matches cyclone exactly)
//! let root = db.root();
//!
//! // Deactivate an entry
//! db.deactivate(serial_number);
//! let new_root = db.root();
//! ```
//!
//! # Compatibility Guarantees
//!
//! When using [`CycloneMode`], the following are guaranteed to match cyclone:
//!
//! - State root computation for any sequence of operations
//! - Twig root hashes
//! - Active bits tree hashes (MTL1, MTL2, MTL3)
//! - Entry tree (left_root) hashes
//!
//! # Feature Flag
//!
//! This module is only available when the `cyclone-compat` feature is enabled.
//! The feature adds a dependency on the `blake3` crate.
//!
//! # See Also
//!
//! - [`CycloneCompatible`] - Trait for compatibility modes
//! - [`CycloneHasher`](crate::mmr::cyclone_hasher::CycloneHasher) - Blake3 level-keyed hasher
//! - [`Twig`](crate::bitmap::cyclone_twig::Twig) - Cyclone twig structure
```

### Step 3: Document Public Types

Add comprehensive documentation to all public types:

```rust
/// A cyclone-compatible authenticated database.
///
/// This type provides the same API as commonware's native QMDB but computes
/// state roots that match cyclone's implementation exactly.
///
/// # Type Parameters
///
/// * `M` - The compatibility mode. Use [`CycloneMode`] for standard cyclone compatibility.
///
/// # Example
///
/// ```rust,ignore
/// use commonware_storage::qmdb::{CycloneDb, CycloneMode};
///
/// let mut db = CycloneDb::<CycloneMode>::new();
///
/// // Add an entry and get its serial number
/// let entry_hash = blake3::hash(b"my entry").into();
/// let sn = db.add(entry_hash);
///
/// // Compute the state root
/// let root = db.root();
/// println!("State root: {:?}", root);
/// ```
///
/// # Thread Safety
///
/// `CycloneDb` is `Send` but not `Sync`. For concurrent access, wrap in a mutex
/// or use separate instances per thread.
pub struct CycloneDb<M: CycloneCompatible = CycloneMode> { ... }
```

### Step 4: Add Examples

**File**: `storage/examples/cyclone_compat.rs`

```rust
//! Example demonstrating cyclone-compatible state root computation.
//!
//! Run with: cargo run --example cyclone_compat --features cyclone-compat

use commonware_storage::qmdb::{CycloneDb, CycloneMode};

fn main() {
    println!("Cyclone QMDB Compatibility Example\n");

    // Create a new cyclone-compatible database
    let mut db = CycloneDb::<CycloneMode>::new();

    // Add some entries
    println!("Adding entries...");
    for i in 0..10 {
        let entry_hash = [i as u8; 32];
        let sn = db.add(entry_hash);
        println!("  Added entry {} with serial number {}", i, sn);
    }

    // Compute state root
    let root = db.root();
    println!("\nState root: 0x{}", hex::encode(root));

    // Deactivate an entry
    println!("\nDeactivating entry 5...");
    db.deactivate(5);

    // Compute new state root
    let new_root = db.root();
    println!("New state root: 0x{}", hex::encode(new_root));

    println!("\nRoot changed: {}", root != new_root);
}
```

### Step 5: Update lib.rs Exports

**File**: `storage/src/lib.rs`

```rust
//! Commonware Storage
//!
//! ...existing docs...
//!
//! # Feature Flags
//!
//! - `cyclone-compat`: Enable cyclone QMDB compatibility mode. See [`qmdb::cyclone_compat`]
//!   for details.

#[cfg(feature = "cyclone-compat")]
#[cfg_attr(docsrs, doc(cfg(feature = "cyclone-compat")))]
pub use qmdb::cyclone_compat::{CycloneCompatible, CycloneDb, CycloneMode};
```

## Files to Create

- `storage/examples/cyclone_compat.rs`

## Files to Modify

- `storage/Cargo.toml` (feature documentation)
- `storage/src/lib.rs` (re-exports and feature docs)
- `storage/src/qmdb/cyclone_compat.rs` (module and type docs)
- `storage/src/mmr/cyclone_hasher.rs` (type docs)
- `storage/src/bitmap/cyclone_twig.rs` (type docs)

## Testing Strategy

### Documentation Tests

Ensure all doc examples compile and run:

```bash
cargo test --doc --features cyclone-compat
```

### Example Runs

```bash
cargo run --example cyclone_compat --features cyclone-compat
```

## Dependencies

- Tasks 2-6 (all implementation tasks)

## Estimated Effort

- 0.5-1 day for documentation
- 0.5 day for examples and testing
