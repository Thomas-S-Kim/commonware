# Task 03b: Hierarchical Bitmap Structure

## Goal

Implement the 3-level hierarchical bitmap for tracking active/inactive entries.

This is the "right subtree" of a segment, combined with the entry tree at level 11.

## Structure

```
Bitmap Root (level 10)
├── L2[0] (level 9)
│   ├── L1[0] (level 8): H_8(page0, page1)
│   └── L1[1] (level 8): H_8(page2, page3)
└── L2[1] (level 9)
    ├── L1[2] (level 8): H_8(page4, page5)
    └── L1[3] (level 8): H_8(page6, page7)
```

- **8 pages**: 32 bytes each = 256 bits per page
- **2048 bits total**: One bit per entry (matching ENTRIES_PER_SEGMENT)
- **3 levels of hashing**: L1 (4 nodes), L2 (2 nodes), L3 (1 node = root)

## Execution

### Step 1: HierarchicalBitmap

**File**: `storage/src/bitmap/hierarchical.rs`

```rust
//! Hierarchical bitmap for balanced tree scheme.
//!
//! 2048 bits organized as 8 x 32-byte pages, hashed as a 3-level tree.
//! Used for tracking active/inactive entries in a segment.

use crate::hasher::level_keyed::Hash32;
use crate::mmr::hasher::Hasher;
use thiserror::Error;

pub const BITMAP_PAGES: usize = 8;
pub const BYTES_PER_PAGE: usize = 32;
pub const BITS_PER_PAGE: usize = BYTES_PER_PAGE * 8; // 256
pub const BITMAP_BYTES: usize = BITMAP_PAGES * BYTES_PER_PAGE; // 256
pub const BITMAP_BITS: usize = BITMAP_BYTES * 8; // 2048

#[derive(Error, Debug)]
pub enum BitmapError {
    #[error("bit index out of bounds: {0} >= {1}")]
    BitOutOfBounds(u32, u32),
    #[error("page index out of bounds: {0} >= {1}")]
    PageOutOfBounds(usize, usize),
}

/// 2048-bit hierarchical bitmap.
///
/// Organized as 8 x 32-byte pages, hashed into a 3-level Merkle tree
/// at levels 8, 9, and 10.
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct HierarchicalBitmap {
    /// Raw bitmap bytes (256 bytes = 2048 bits)
    pages: [u8; BITMAP_BYTES],
    /// Level 8: Hash of page pairs (4 nodes)
    l1: [Hash32; 4],
    /// Level 9: Hash of L1 pairs (2 nodes)
    l2: [Hash32; 2],
    /// Level 10: Bitmap root (1 node)
    root: Hash32,
    /// Whether hashes need recomputation
    dirty: bool,
}

impl Default for HierarchicalBitmap {
    fn default() -> Self {
        Self::new()
    }
}

impl HierarchicalBitmap {
    /// Create a new bitmap with all bits cleared (inactive).
    pub const fn new() -> Self {
        Self {
            pages: [0u8; BITMAP_BYTES],
            l1: [Hash32([0u8; 32]); 4],
            l2: [Hash32([0u8; 32]); 2],
            root: Hash32([0u8; 32]),
            dirty: true,
        }
    }

    /// Set bit at index (mark entry as active).
    pub fn set_bit(&mut self, index: u32) -> Result<(), BitmapError> {
        if index >= BITMAP_BITS as u32 {
            return Err(BitmapError::BitOutOfBounds(index, BITMAP_BITS as u32));
        }
        let byte_idx = (index / 8) as usize;
        let bit_idx = index % 8;
        self.pages[byte_idx] |= 1 << bit_idx;
        self.dirty = true;
        Ok(())
    }

    /// Clear bit at index (mark entry as inactive).
    pub fn clear_bit(&mut self, index: u32) -> Result<(), BitmapError> {
        if index >= BITMAP_BITS as u32 {
            return Err(BitmapError::BitOutOfBounds(index, BITMAP_BITS as u32));
        }
        let byte_idx = (index / 8) as usize;
        let bit_idx = index % 8;
        self.pages[byte_idx] &= !(1 << bit_idx);
        self.dirty = true;
        Ok(())
    }

    /// Get bit at index.
    pub fn get_bit(&self, index: u32) -> Result<bool, BitmapError> {
        if index >= BITMAP_BITS as u32 {
            return Err(BitmapError::BitOutOfBounds(index, BITMAP_BITS as u32));
        }
        let byte_idx = (index / 8) as usize;
        let bit_idx = index % 8;
        Ok((self.pages[byte_idx] >> bit_idx) & 1 == 1)
    }

    /// Get page as 32-byte slice.
    pub fn page(&self, idx: usize) -> Result<&[u8; BYTES_PER_PAGE], BitmapError> {
        if idx >= BITMAP_PAGES {
            return Err(BitmapError::PageOutOfBounds(idx, BITMAP_PAGES));
        }
        let start = idx * BYTES_PER_PAGE;
        let end = start + BYTES_PER_PAGE;
        Ok(self.pages[start..end].try_into().unwrap())
    }

    /// Get the 32-byte block containing a specific bit.
    /// Used for proof generation.
    pub fn block_for_index(&self, index: u32) -> Result<&[u8; 32], BitmapError> {
        let page_idx = (index / BITS_PER_PAGE as u32) as usize;
        self.page(page_idx)
    }

    /// Compute bitmap tree and return root.
    pub fn compute_root<H: Hasher<Hash32>>(&mut self, hasher: &mut H) -> Hash32 {
        if !self.dirty {
            return self.root;
        }

        // Level 8: Hash page pairs
        for i in 0..4 {
            let left_page = &self.pages[i * 2 * BYTES_PER_PAGE..(i * 2 + 1) * BYTES_PER_PAGE];
            let right_page = &self.pages[(i * 2 + 1) * BYTES_PER_PAGE..(i * 2 + 2) * BYTES_PER_PAGE];

            // Pages are treated as Hash32 for hashing
            let left = Hash32(left_page.try_into().unwrap());
            let right = Hash32(right_page.try_into().unwrap());

            let pos = (1u64 << 8) - 1 + i as u64;
            self.l1[i] = hasher.node_digest(pos, &left, &right);
        }

        // Level 9: Hash L1 pairs
        for i in 0..2 {
            let pos = (1u64 << 9) - 1 + i as u64;
            self.l2[i] = hasher.node_digest(pos, &self.l1[i * 2], &self.l1[i * 2 + 1]);
        }

        // Level 10: Bitmap root
        let pos = (1u64 << 10) - 1;
        self.root = hasher.node_digest(pos, &self.l2[0], &self.l2[1]);

        self.dirty = false;
        self.root
    }

    /// Get cached root (may be stale if dirty).
    pub fn cached_root(&self) -> Hash32 {
        self.root
    }

    /// Get L1 nodes (for proof generation).
    pub fn l1(&self) -> &[Hash32; 4] {
        &self.l1
    }

    /// Get L2 nodes (for proof generation).
    pub fn l2(&self) -> &[Hash32; 2] {
        &self.l2
    }

    /// Check if hashes need recomputation.
    pub fn is_dirty(&self) -> bool {
        self.dirty
    }
}
```

### Step 2: Bitmap Path for Proofs

```rust
/// Build sibling path through bitmap tree for entry at given index.
///
/// Returns 3 sibling hashes (levels 8, 9, 10) for the bitmap portion of proof.
pub fn build_bitmap_path(bitmap: &HierarchicalBitmap, entry_index: u16) -> [Hash32; 3] {
    let page_idx = (entry_index / BITS_PER_PAGE as u16) as usize;

    // Level 8 sibling: the other page in the pair
    let l1_sibling_idx = page_idx ^ 1;
    let l1_sibling = if l1_sibling_idx < BITMAP_PAGES {
        Hash32(*bitmap.page(l1_sibling_idx).unwrap())
    } else {
        Hash32::default()
    };

    // Level 9 sibling: the other L1 in the pair
    let l1_pair_idx = page_idx / 2;
    let l2_sibling_idx = l1_pair_idx ^ 1;
    let l2_sibling = bitmap.l1()[l2_sibling_idx];

    // Level 10 sibling: the other L2
    let l2_pair_idx = l1_pair_idx / 2;
    let l3_sibling_idx = l2_pair_idx ^ 1;
    let l3_sibling = bitmap.l2()[l3_sibling_idx];

    [l1_sibling, l2_sibling, l3_sibling]
}

/// Verify bitmap path from page to bitmap root.
pub fn verify_bitmap_path<H: Hasher<Hash32>>(
    page: &[u8; 32],
    page_idx: usize,
    path: &[Hash32; 3],
    expected_root: &Hash32,
    hasher: &mut H,
) -> bool {
    // Start with page as Hash32
    let mut current = Hash32(*page);
    let mut idx = page_idx;

    // Level 8: hash with sibling page
    let pos8 = (1u64 << 8) - 1 + (idx / 2) as u64;
    current = if idx % 2 == 0 {
        hasher.node_digest(pos8, &current, &path[0])
    } else {
        hasher.node_digest(pos8, &path[0], &current)
    };
    idx /= 2;

    // Level 9: hash with L1 sibling
    let pos9 = (1u64 << 9) - 1 + (idx / 2) as u64;
    current = if idx % 2 == 0 {
        hasher.node_digest(pos9, &current, &path[1])
    } else {
        hasher.node_digest(pos9, &path[1], &current)
    };
    idx /= 2;

    // Level 10: hash with L2 sibling
    let pos10 = (1u64 << 10) - 1;
    current = if idx % 2 == 0 {
        hasher.node_digest(pos10, &current, &path[2])
    } else {
        hasher.node_digest(pos10, &path[2], &current)
    };

    current == *expected_root
}
```

### Step 3: Module Export

**File**: `storage/src/bitmap/mod.rs`

```rust
#[cfg(feature = "balanced_tree")]
pub mod hierarchical;

#[cfg(feature = "balanced_tree")]
pub use hierarchical::{
    HierarchicalBitmap, BitmapError,
    BITMAP_PAGES, BYTES_PER_PAGE, BITS_PER_PAGE, BITMAP_BITS,
    build_bitmap_path, verify_bitmap_path,
};
```

## Interface Impact

| Interface | Change |
|-----------|--------|
| `HierarchicalBitmap` | **New struct** |
| `BitmapError` | **New error type** |
| `build_bitmap_path()` | **New function** |
| `verify_bitmap_path()` | **New function** |

## Testing

```rust
#[test]
fn test_bitmap_set_get_clear() {
    let mut bitmap = HierarchicalBitmap::new();

    // Test boundary values
    for idx in [0, 255, 256, 1000, 2047] {
        assert!(!bitmap.get_bit(idx).unwrap());
        bitmap.set_bit(idx).unwrap();
        assert!(bitmap.get_bit(idx).unwrap());
        bitmap.clear_bit(idx).unwrap();
        assert!(!bitmap.get_bit(idx).unwrap());
    }
}

#[test]
fn test_bitmap_bounds_checking() {
    let mut bitmap = HierarchicalBitmap::new();
    assert!(bitmap.set_bit(2048).is_err());
    assert!(bitmap.get_bit(2048).is_err());
    assert!(bitmap.page(8).is_err());
}

#[test]
fn test_bitmap_tree_vectors() {
    let vectors = load_test_vectors("bitmap_hashing.toml");

    for v in vectors.bitmap_tree {
        let mut bitmap = HierarchicalBitmap::new();
        for (i, page) in v.pages.iter().enumerate() {
            for (j, byte) in page.iter().enumerate() {
                bitmap.pages[i * 32 + j] = *byte;
            }
        }

        let mut hasher = LevelKeyed::default();
        let root = bitmap.compute_root(&mut hasher);

        assert_eq!(bitmap.l1(), &v.l1, "L1 mismatch: {}", v.name);
        assert_eq!(bitmap.l2(), &v.l2, "L2 mismatch: {}", v.name);
        assert_eq!(root, v.bitmap_root, "Root mismatch: {}", v.name);
    }
}

#[test]
fn test_bitmap_path_roundtrip() {
    let mut bitmap = HierarchicalBitmap::new();
    bitmap.set_bit(0).unwrap();
    bitmap.set_bit(500).unwrap();
    bitmap.set_bit(2000).unwrap();

    let mut hasher = LevelKeyed::default();
    let root = bitmap.compute_root(&mut hasher);

    // Verify path for each page
    for page_idx in 0..8 {
        let page = bitmap.page(page_idx).unwrap();
        let path = build_bitmap_path(&bitmap, (page_idx * 256) as u16);

        assert!(
            verify_bitmap_path(page, page_idx, &path, &root, &mut hasher),
            "Path verification failed for page {}", page_idx
        );
    }
}
```

## Dependencies

- Task 01b (bitmap test vectors)
- Task 02 (LevelKeyed hasher)
- Milestone 1 complete
