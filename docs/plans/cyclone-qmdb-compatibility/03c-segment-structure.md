# Task 03c: Full Segment Structure (Entry Tree + Bitmap)

**Milestone 2 Task** - Requires Milestone 1 complete.

## Current State

From Milestone 1:
- `EntryTree` (Task 03a): Binary tree of 2048 entry hashes (levels 0-11)
- `HierarchicalBitmap` (Task 03b): 3-level bitmap tree (levels 8-10)

## Goal

Combine entry tree and bitmap into a complete `SegmentState` that:
1. Stores entry hashes (2048 slots)
2. Tracks active/inactive bits
3. Computes segment root = H_11(entry_root, bitmap_root) → level 12
4. Implements `ChunkState<D>` trait (Task 05b)

This is the **full segment** used in the unified `Qmdb<S>` interface.

## Structure

```
Segment Root (level 12) = H_11(entry_root, bitmap_root)
├── Entry Tree Root (level 11, left child)
│   └── Binary tree of 2048 entry hashes (levels 0-10)
└── Bitmap Tree Root (level 10, right child)
    ├── L2[0] (level 9)
    │   ├── L1[0] (level 8): hash(page0, page1)
    │   └── L1[1] (level 8): hash(page2, page3)
    └── L2[1] (level 9)
        ├── L1[2] (level 8): hash(page4, page5)
        └── L1[3] (level 8): hash(page6, page7)
```

**Level numbering:**
- Entry tree: 2048 leaves at level 0, root at level 11 (11 levels for 2^11 entries)
- Bitmap tree: Pages hashed at level 8, root at level 10 (3 levels)
- Segment root: Computed using level 11 hash, result placed at level 12
- Upper tree: Starts at level 13 (FIRST_LEVEL_ABOVE_TWIG)

## Execution

### Step 1: Constants

**File**: `storage/src/bitmap/hierarchical.rs`

```rust
pub const ENTRIES_PER_SEGMENT: u32 = 2048;
pub const SEGMENT_SHIFT: u32 = 11;  // 2^11 = 2048
pub const SEGMENT_MASK: u32 = ENTRIES_PER_SEGMENT - 1;

pub const BITMAP_PAGES: usize = 8;
pub const BYTES_PER_PAGE: usize = 32;
pub const BITMAP_BYTES: usize = BITMAP_PAGES * BYTES_PER_PAGE; // 256
```

### Step 2: Hierarchical Bitmap

```rust
use crate::hasher::level_keyed::Hash32;
use thiserror::Error;

#[derive(Error, Debug)]
pub enum BitmapError {
    #[error("bit offset out of bounds: {0} >= {1}")]
    BitOutOfBounds(u32, u32),
    #[error("page index out of bounds: {0} >= {1}")]
    PageOutOfBounds(usize, usize),
}

/// 2048-bit bitmap organized as 8 x 32-byte pages.
///
/// Hashed as a 3-level binary tree at levels 8, 9, 10.
/// All operations perform bounds checking for adversarial safety.
#[derive(Clone, Debug, Default, PartialEq, Eq)]
pub struct HierarchicalBitmap(pub [u8; BITMAP_BYTES]);

impl HierarchicalBitmap {
    pub const fn new() -> Self { Self([0u8; BITMAP_BYTES]) }

    /// Set bit at offset. Returns error if offset >= ENTRIES_PER_SEGMENT.
    pub fn set_bit(&mut self, offset: u32) -> Result<(), BitmapError> {
        if offset >= ENTRIES_PER_SEGMENT {
            return Err(BitmapError::BitOutOfBounds(offset, ENTRIES_PER_SEGMENT));
        }
        let byte_idx = (offset / 8) as usize;
        let bit_idx = offset % 8;
        self.0[byte_idx] |= 1 << bit_idx;
        Ok(())
    }

    /// Clear bit at offset. Returns error if offset >= ENTRIES_PER_SEGMENT.
    pub fn clear_bit(&mut self, offset: u32) -> Result<(), BitmapError> {
        if offset >= ENTRIES_PER_SEGMENT {
            return Err(BitmapError::BitOutOfBounds(offset, ENTRIES_PER_SEGMENT));
        }
        let byte_idx = (offset / 8) as usize;
        let bit_idx = offset % 8;
        self.0[byte_idx] &= !(1 << bit_idx);
        Ok(())
    }

    /// Get bit at offset. Returns error if offset >= ENTRIES_PER_SEGMENT.
    pub fn get_bit(&self, offset: u32) -> Result<bool, BitmapError> {
        if offset >= ENTRIES_PER_SEGMENT {
            return Err(BitmapError::BitOutOfBounds(offset, ENTRIES_PER_SEGMENT));
        }
        let byte_idx = (offset / 8) as usize;
        let bit_idx = offset % 8;
        Ok((self.0[byte_idx] >> bit_idx) & 1 == 1)
    }

    /// Get page as 32-byte slice for hashing.
    pub fn page(&self, idx: usize) -> Result<&[u8; BYTES_PER_PAGE], BitmapError> {
        if idx >= BITMAP_PAGES {
            return Err(BitmapError::PageOutOfBounds(idx, BITMAP_PAGES));
        }
        Ok(self.0[idx * BYTES_PER_PAGE..(idx + 1) * BYTES_PER_PAGE]
            .try_into()
            .expect("slice length is BYTES_PER_PAGE"))
    }
}
```

### Step 3: Segment Structure

```rust
use crate::mmr::hasher::Hasher;
use crate::hasher::level_keyed::Hash32;

/// Intermediate hashes for a segment.
///
/// Stores bitmap tree levels and combined segment root.
/// Used internally by SegmentState (Task 5).
#[derive(Clone, Debug, Default, PartialEq, Eq)]
pub struct Segment {
    /// Level 8: hash pairs of bitmap pages
    pub bitmap_l1: [Hash32; 4],
    /// Level 9: hash pairs of L1
    pub bitmap_l2: [Hash32; 2],
    /// Level 10: bitmap tree root
    pub bitmap_root: Hash32,
    /// Root of entry hash tree (level 11 left child)
    pub entry_root: Hash32,
    /// Combined root (level 11)
    pub segment_root: Hash32,
}

impl Segment {
    /// Sync bitmap tree from raw bitmap using provided hasher.
    ///
    /// # Errors
    /// Returns error if bitmap page access fails (should not happen with valid bitmap).
    pub fn sync_bitmap<H: Hasher<Hash32>>(
        &mut self,
        bitmap: &HierarchicalBitmap,
        hasher: &mut H,
    ) -> Result<(), BitmapError> {
        // Level 8: hash page pairs
        for i in 0..4 {
            let left = bitmap.page(i * 2)?;
            let right = bitmap.page(i * 2 + 1)?;
            // Position encoding for level 8 nodes
            let pos = (1 << 8) - 1 + i as u64;
            self.bitmap_l1[i] = hasher.node_digest(pos, &Hash32(*left), &Hash32(*right));
        }

        // Level 9: hash L1 pairs
        for i in 0..2 {
            let pos = (1 << 9) - 1 + i as u64;
            self.bitmap_l2[i] = hasher.node_digest(pos, &self.bitmap_l1[i * 2], &self.bitmap_l1[i * 2 + 1]);
        }

        // Level 10: bitmap root
        let pos = (1 << 10) - 1;
        self.bitmap_root = hasher.node_digest(pos, &self.bitmap_l2[0], &self.bitmap_l2[1]);
        Ok(())
    }

    /// Compute segment root from entry root and bitmap root.
    /// Uses level 11 hash (children level), result is segment_root at level 12.
    pub fn sync_root<H: Hasher<Hash32>>(&mut self, hasher: &mut H) {
        // Position encoding gives level 11 for domain separation
        // entry_root (level 11) + bitmap_root (level 10) -> segment_root (level 12)
        let pos = (1u64 << 11) - 1;
        self.segment_root = hasher.node_digest(pos, &self.entry_root, &self.bitmap_root);
    }

    /// Full sync: bitmap tree then segment root.
    pub fn sync_all<H: Hasher<Hash32>>(
        &mut self,
        bitmap: &HierarchicalBitmap,
        hasher: &mut H,
    ) -> Result<(), BitmapError> {
        self.sync_bitmap(bitmap, hasher)?;
        self.sync_root(hasher);
        Ok(())
    }
}
```

### Step 4: Export

**File**: `storage/src/bitmap/mod.rs`

```rust
#[cfg(feature = "balanced_tree")]
pub mod hierarchical;
#[cfg(feature = "balanced_tree")]
pub use hierarchical::{
    BitmapError, HierarchicalBitmap, Segment,
    ENTRIES_PER_SEGMENT, BITMAP_PAGES, BYTES_PER_PAGE,
};
```

## Interface Impact

| Interface | Change |
|-----------|--------|
| `BitMap<D,N,S>` | **No change** |
| `HierarchicalBitmap` | **New struct** (pub(crate)) |
| `Segment` | **New struct** (pub(crate)) |
| `BitmapError` | **New error type** |

These structs are wrapped by `SegmentState` which implements the `ChunkState` trait.

## Files to Create

- `storage/src/bitmap/hierarchical.rs`

## Files to Modify

- `storage/src/bitmap/mod.rs`

## Testing

```rust
#[test]
fn test_bitmap_set_get_clear() {
    let mut bits = HierarchicalBitmap::new();
    for i in [0, 100, 1000, 2047] {
        assert!(!bits.get_bit(i).unwrap());
        bits.set_bit(i).unwrap();
        assert!(bits.get_bit(i).unwrap());
        bits.clear_bit(i).unwrap();
        assert!(!bits.get_bit(i).unwrap());
    }
}

#[test]
fn test_bitmap_bounds_checking() {
    let mut bits = HierarchicalBitmap::new();
    assert!(bits.set_bit(2048).is_err());
    assert!(bits.get_bit(2048).is_err());
    assert!(bits.page(8).is_err());
}

#[test]
fn test_segment_uses_hasher_trait() {
    let mut segment = Segment::default();
    let bitmap = HierarchicalBitmap::new();
    let mut hasher = LevelKeyed::default();
    segment.sync_all(&bitmap, &mut hasher).unwrap();
    // Verify against test vector
}
```

## Dependencies

- Task 02 (LevelKeyed hasher)
- Task 03a (EntryTree - Milestone 1)
- Task 03b (HierarchicalBitmap - Milestone 2)
- Milestone 1 complete (core hashing validated)
