# Task 01b: Bitmap Hashing Test Vectors

## Goal

Generate test vectors for **bitmap/ActiveBits hashing** (Milestone 2):
- Hierarchical bitmap tree (8 pages, levels 8-10)
- Segment root (entry_root + bitmap_root combined at level 11 → result at level 12)
- Full state roots with active bits

These vectors build on Milestone 1 (core hashing must pass first).

## Test Vectors to Generate

### 1. Bitmap Page Hashing

```toml
[[bitmap_page_hash]]
name = "all_zeros"
page0 = "0x00...00" # 32 bytes
page1 = "0x00...00"
level = 8
expected = "0x..." # H_8(page0, page1)

[[bitmap_page_hash]]
name = "all_ones"
page0 = "0xff...ff"
page1 = "0xff...ff"
level = 8
expected = "0x..."

[[bitmap_page_hash]]
name = "mixed"
page0 = "0xaa..."
page1 = "0xbb..."
level = 8
expected = "0x..."
```

### 2. Bitmap Tree Vectors (3-Level)

```toml
[[bitmap_tree]]
name = "all_inactive"
# 8 pages of 32 bytes each (256 bits per page, 2048 total)
pages = ["0x00...00", "0x00...00", "0x00...00", "0x00...00",
         "0x00...00", "0x00...00", "0x00...00", "0x00...00"]
# Level 8: 4 nodes (pairs of pages hashed)
l1 = ["0x...", "0x...", "0x...", "0x..."]
# Level 9: 2 nodes (pairs of L1 hashed)
l2 = ["0x...", "0x..."]
# Level 10: 1 node (bitmap root)
bitmap_root = "0x..."

[[bitmap_tree]]
name = "all_active"
pages = ["0xff...ff", ...] # All bits set
l1 = [...]
l2 = [...]
bitmap_root = "0x..."

[[bitmap_tree]]
name = "first_256_active"
# First page all 1s, rest all 0s
pages = ["0xff...ff", "0x00...00", ...]
l1 = [...]
l2 = [...]
bitmap_root = "0x..."

[[bitmap_tree]]
name = "sparse_active"
# Random pattern
active_indices = [0, 100, 500, 1000, 2000]
pages = [...] # Computed from indices
bitmap_root = "0x..."
```

### 3. Segment Root Vectors

```toml
[[segment_root]]
name = "null_segment"
entry_root = "0x..." # From null entry tree (Milestone 1)
bitmap_root = "0x..." # From all-inactive bitmap
segment_root = "0x..." # H_11(entry_root, bitmap_root) -> level 12

[[segment_root]]
name = "with_one_active_entry"
entries = ["0xaa..."] # Single entry
active_indices = [0]
entry_root = "0x..."
bitmap_root = "0x..."
segment_root = "0x..."

[[segment_root]]
name = "with_deactivated_entry"
entries = ["0xaa...", "0xbb..."]
active_indices = [0] # Entry 1 deactivated
entry_root = "0x..."
bitmap_root = "0x..."
segment_root = "0x..."
```

### 4. Full State Root Vectors

```toml
[[full_state_root]]
name = "single_active_entry"
segments = [
    { entries = ["0x..."], active_indices = [0] }
]
state_root = "0x..."

[[full_state_root]]
name = "multiple_segments_mixed"
segments = [
    { entries = [...], active_indices = [0, 5, 100] },
    { entries = [...], active_indices = [0, 1, 2] },
]
state_root = "0x..."

[[full_state_root]]
name = "deactivation_changes_root"
# Same entries, different active sets
base_entries = [...]
# State 1: all active
active_1 = [0, 1, 2, 3]
state_root_1 = "0x..."
# State 2: entry 2 deactivated
active_2 = [0, 1, 3]
state_root_2 = "0x..."
```

### 5. Reference Null Hashes (Bitmap-Related)

```toml
[null_bitmap]
# Precomputed from QMDB reference
null_page = "0x00...00" # 32 zero bytes
null_l1 = "0x..." # H_8(null_page, null_page)
null_l2 = "0x..." # H_9(null_l1, null_l1)
null_bitmap_root = "0x..." # H_10(null_l2, null_l2)
null_segment_root = "0x..." # H_11(null_entry_root, null_bitmap_root)
```

## Execution

### Step 1: Create Vector Generator

**File**: `scripts/generate_bitmap_vectors.rs`

```rust
use qmdb_common::merkletree::twig::Twig;
use qmdb_common::merkletree::activebits::ActiveBits;
use qmdb_common::merkletree::hash::merkle_node_hash_inplace;

fn generate_bitmap_tree_vectors() -> Vec<BitmapTreeVector> {
    let mut vectors = Vec::new();

    // All inactive
    let mut twig = Twig::default();
    let bits = ActiveBits::new();
    twig.sync_l1(&bits);
    twig.sync_l2();
    twig.sync_l3();
    vectors.push(BitmapTreeVector {
        name: "all_inactive".into(),
        pages: bits.as_pages(),
        l1: twig.active_bits_mtl1.clone(),
        l2: twig.active_bits_mtl2.clone(),
        bitmap_root: twig.active_bits_mtl3,
    });

    // All active
    let mut bits = ActiveBits::new();
    for i in 0..2048 {
        bits.set(i);
    }
    let mut twig = Twig::default();
    twig.sync_l1(&bits);
    twig.sync_l2();
    twig.sync_l3();
    vectors.push(BitmapTreeVector {
        name: "all_active".into(),
        pages: bits.as_pages(),
        l1: twig.active_bits_mtl1.clone(),
        l2: twig.active_bits_mtl2.clone(),
        bitmap_root: twig.active_bits_mtl3,
    });

    // ... more patterns

    vectors
}

fn generate_segment_root_vectors() -> Vec<SegmentRootVector> {
    // Generate using QMDB's actual Twig implementation
    // to ensure exact compatibility
}
```

### Step 2: Generate from QMDB Reference

```bash
cd /path/to/qmdb
cargo run --bin generate_vectors -- \
    --type bitmap \
    --output vectors/bitmap_hashing.toml
```

### Step 3: Copy to Commonware

```bash
cp vectors/bitmap_hashing.toml \
   /path/to/commonware/storage/src/qmdb/tests/fixtures/
```

## Output Location

```
storage/src/qmdb/tests/fixtures/
├── core_hashing.toml      # Milestone 1 vectors
└── bitmap_hashing.toml    # This task's output
```

## Success Criteria

- [ ] Bitmap page hash vectors (8 pairs)
- [ ] Bitmap tree vectors: all_inactive, all_active, sparse patterns
- [ ] Segment root vectors: null, with entries, with deactivation
- [ ] Full state root vectors: single segment, multiple segments
- [ ] Null bitmap hashes match QMDB reference
- [ ] All vectors generated from QMDB reference implementation

## Dependencies

- Task 01a complete (core hashing vectors)
- Milestone 1 complete (core hashing implementation)
