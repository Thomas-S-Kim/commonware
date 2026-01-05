# Grafting Analysis: MMR vs Balanced Tree

## Executive Summary

The balanced tree scheme does **not** need the complex position-mapping grafting of MMR. Instead, it uses fixed structural combination at specific levels. This analysis shows how to support both schemes with minimal changes.

**Key Finding**: The balanced tree's "grafting" is simpler - it's just binary tree hashing at fixed levels. We should **not** try to force it into the MMR grafting abstraction. Instead, the `StateRootScheme::compute_root()` method (already in Task 05) handles the structural differences.

## Current MMR Grafting

### How It Works

MMR grafting combines a "peak tree" (bitmap) with a "base MMR" (operations log):

```
Base MMR (8 operations):        Peak Tree (2 segments):
    Height                          Height
      3              14               1              2
                   /    \                          /   \
      2        6            13        0        0       1
             /   \        /    \
      1     2     5      9     12
           / \   / \    / \   /  \
      0   0   1 3   4  7   8 10  11

Grafting (height=2):
- Peak leaf 0 -> Base node 6
- Peak leaf 1 -> Base node 13
```

### Key Components

1. **`destination_pos(pos, height)`**: Maps peak tree position to base MMR position
2. **`source_pos(pos, height)`**: Inverse mapping
3. **`Hasher`**: Computes leaf digest as `H(element || grafted_digest)`
4. **`Storage`**: Routes `get_node()` based on height threshold
5. **`Verifier`**: Reconstructs digests at grafting boundary during verification

### Position Mapping Algorithm

```rust
// Walk down corresponding branches of peak and base trees
fn destination_pos(peak_pos: Position, height: u32) -> Position {
    // Complex bit manipulation to find corresponding position
    // in base MMR after walking up `height` levels
}
```

## Balanced Tree "Grafting"

### How It Works

The balanced tree has a **fixed structure** - no position mapping needed:

```
Segment (Twig) Structure:
Level 11:  segment_root = H_11(entry_root, bitmap_root)  <-- GRAFT POINT
           /                        \
Level 11: entry_root               bitmap_root (L10)
          |                        /          \
Levels 0-10: 2048 entry hashes   L9[0]       L9[1]
                                  |   |       |   |
                                L8[0-3]     L8[0-3]
                                  |           |
                              256 bits    256 bits

Upper Tree (above segments):
Level 12+: Binary tree combining segment_roots
```

### Key Differences from MMR

| Aspect | MMR Grafting | Balanced Tree |
|--------|--------------|---------------|
| Position mapping | Dynamic via `destination_pos()` | Fixed levels (11, 12+) |
| Domain separation | Position in hash input | Level XOR'd into Blake3 IV |
| Structure | Variable (MMR peaks) | Fixed (2048 entries/segment) |
| Leaf digest | `H(element \|\| grafted_digest)` | Level-keyed: `H_level(left, right)` |
| Root computation | Peak bagging with size | Virtual balanced tree |

### Critical Insight

The balanced tree's "grafting" is simply:
1. **Level 11**: `segment_root = level_keyed_hash(11, entry_root, bitmap_root)`
2. **Level 12+**: `parent = level_keyed_hash(level, left_segment_root, right_segment_root)`

This is **standard binary tree hashing** with level-keyed domain separation - no complex position mapping required.

## Analysis: What Needs to Change

### Option A: Abstract Grafting Trait (NOT RECOMMENDED)

```rust
// This would over-engineer the solution
pub trait Grafting<D: Digest>: Send + Sync {
    type GraftingStrategy;
    fn graft_leaf(&mut self, pos: Position, element: &[u8], grafted: &D) -> D;
    fn graft_node(&mut self, pos: Position, left: &D, right: &D) -> D;
}
```

**Problem**: Forces the balanced tree into an abstraction that doesn't fit its simpler structure.

### Option B: Scheme-Specific Root Computation (RECOMMENDED)

The `StateRootScheme::compute_root()` from Task 05 already handles this:

```rust
pub trait StateRootScheme: Send + Sync + Clone + 'static {
    // ... other methods ...

    /// Compute final state root from chunk/segment roots.
    /// MMR: Peak bagging with grafted bitmap
    /// Balanced tree: Virtual tree overlay
    fn compute_root(
        hasher: &mut Self::Hasher,
        chunk_roots: impl Iterator<Item = (u64, Self::Digest)>,
    ) -> Self::Digest;
}
```

**Why this works**:
- MMR scheme uses existing grafting code internally
- Balanced tree scheme uses `BalancedTreeRootBuilder` internally
- No new abstraction needed - each scheme implements root computation differently

## Detailed Changes Required

### For Balanced Tree Scheme: NO grafting.rs changes needed

The balanced tree scheme handles tree combination internally:

#### 1. Segment Internal Structure (Task 03)

```rust
// storage/src/bitmap/hierarchical.rs
impl Segment {
    /// Combine entry_root and bitmap_root at level 11
    pub fn sync_root<H: Hasher<Hash32>>(&mut self, hasher: &mut H) {
        let pos = (1u64 << 11) - 1; // Level 11 position encoding
        self.segment_root = hasher.node_digest(pos, &self.entry_root, &self.bitmap_root);
    }
}
```

#### 2. Upper Tree (Task 04)

```rust
// storage/src/qmdb/balanced_tree_root.rs
impl<H: Hasher<Hash32>> BalancedTreeRootBuilder<H> {
    /// Add segment root as leaf at level 12
    pub fn add_segment(&mut self, segment_id: u64, segment_root: Hash32) {
        self.push(StackNode {
            level: SEGMENT_ROOT_LEVEL, // 12
            index: segment_id,
            hash: segment_root,
        });
    }

    /// Merge siblings at same level
    fn try_merge_top(&mut self) -> bool {
        // ... uses level-keyed hashing at level 13, 14, etc.
    }
}
```

#### 3. Scheme Implementation (Task 05)

```rust
// storage/src/qmdb/scheme/balanced_tree.rs
impl StateRootScheme for BalancedTreeScheme {
    fn compute_root(
        hasher: &mut LevelKeyed,
        chunk_roots: impl Iterator<Item = (u64, Hash32)>,
    ) -> Hash32 {
        let mut builder = BalancedTreeRootBuilder::new(hasher.fork());
        for (id, root) in chunk_roots {
            builder.add_segment(id, root);
        }
        builder.finalize()
    }
}
```

### For MMR Scheme: Wrap existing grafting

The MMR scheme continues to use the existing grafting infrastructure:

```rust
// storage/src/qmdb/scheme/mmr.rs
impl StateRootScheme for MmrScheme {
    fn compute_root(
        hasher: &mut StandardHasher<H>,
        chunk_roots: impl Iterator<Item = (u64, H::Digest)>,
    ) -> H::Digest {
        // Use existing grafting::Storage and root computation
        // This is essentially what qmdb/current/mod.rs::root() does today
    }
}
```

## Files to Modify

### NO CHANGES NEEDED:
- `storage/src/mmr/grafting.rs` - Keep as-is for MMR scheme

### ALREADY PLANNED (Tasks 02-05):
- `storage/src/hasher/level_keyed.rs` - LevelKeyed hasher
- `storage/src/bitmap/hierarchical.rs` - Segment with internal tree combination
- `storage/src/qmdb/balanced_tree_root.rs` - Upper tree builder
- `storage/src/qmdb/scheme/balanced_tree.rs` - BalancedTreeScheme::compute_root()
- `storage/src/qmdb/scheme/mmr.rs` - MmrScheme wrapping existing grafting

### MINOR UPDATES NEEDED:

#### `storage/src/qmdb/current/mod.rs`

Current:
```rust
async fn root<E, H, const N: usize>(
    hasher: &mut StandardHasher<H>,
    height: u32,
    status: &CleanBitMap<H::Digest, N>,
    mmr: &Mmr<E, H::Digest, Clean<DigestOf<H>>>,
) -> Result<H::Digest, Error>
```

Updated to use scheme:
```rust
async fn root<S: StateRootScheme>(
    hasher: &mut S::Hasher,
    chunks: &[S::ChunkState],
) -> Result<S::Digest, Error> {
    let chunk_roots = chunks.iter().enumerate().map(|(id, chunk)| {
        (id as u64, chunk.get_root())
    });
    Ok(S::compute_root(hasher, chunk_roots))
}
```

## Proof Generation and Verification

### Balanced Tree Proofs

For the balanced tree, proofs need to include:
1. Entry tree sibling path (levels 0-10)
2. Bitmap tree sibling path (levels 8-10)
3. Upper tree sibling path (levels 12+)

This is simpler than MMR grafting because:
- No dynamic position mapping
- Fixed levels where trees combine
- Standard Merkle proof structure

#### Proof Structure (New)

```rust
// storage/src/qmdb/proof/balanced_tree.rs

/// Proof for a single entry in balanced tree scheme
pub struct BalancedTreeProof {
    /// Entry index within segment (0-2047)
    pub entry_index: u16,
    /// Segment ID
    pub segment_id: u64,
    /// Sibling hashes from entry to entry_root (levels 0-10)
    pub entry_path: Vec<Hash32>,
    /// Sibling hashes from bitmap leaf to bitmap_root (levels 8-10)
    pub bitmap_path: [Hash32; 3],
    /// Sibling hashes from segment_root to state_root (levels 12+)
    pub upper_path: Vec<Hash32>,
    /// The active bits for this entry's 256-bit block
    pub active_bits_block: [u8; 32],
}

impl BalancedTreeProof {
    pub fn verify(
        &self,
        entry_hash: &Hash32,
        expected_root: &Hash32,
        hasher: &mut LevelKeyed,
    ) -> bool {
        // 1. Verify entry path to entry_root
        let entry_root = self.compute_entry_root(entry_hash, hasher);

        // 2. Verify bitmap path to bitmap_root
        let bitmap_root = self.compute_bitmap_root(hasher);

        // 3. Combine at level 11 to get segment_root
        let pos = (1u64 << 11) - 1;
        let segment_root = hasher.node_digest(pos, &entry_root, &bitmap_root);

        // 4. Verify upper path to state_root
        let computed_root = self.compute_upper_root(&segment_root, hasher);

        computed_root == *expected_root
    }
}
```

### MMR Proofs (Unchanged)

The existing `grafting::Verifier` handles MMR proof verification. No changes needed.

## Summary: Minimal Changes

| Component | Change | Reason |
|-----------|--------|--------|
| `grafting.rs` | **None** | MMR scheme uses as-is |
| `LevelKeyed` | **New** (Task 02) | Level-keyed hashing for balanced tree |
| `Segment` | **New** (Task 03) | Internal tree combination at level 11 |
| `BalancedTreeRootBuilder` | **New** (Task 04) | Upper tree at levels 12+ |
| `StateRootScheme` | **New** (Task 05) | Unified trait with proof methods |
| `Proof<S>` | **New** (Task 05) | Unified proof type for all schemes |
| `Qmdb<S>` | **New** (Task 05) | Unified database interface |
| `MmrScheme` | **New** (Task 05) | Wraps existing grafting |
| `BalancedTreeScheme` | **New** (Task 05) | Uses builder internally |
| `current/mod.rs` | **Update** | Use scheme for root computation |

## Key Principle

**Do not abstract grafting**. The schemes are fundamentally different:
- MMR grafting: Complex position mapping between two tree structures
- Balanced tree: Simple fixed-level binary tree hashing

The `StateRootScheme::compute_root()` method is the right abstraction level. Each scheme implements it differently without sharing grafting code.

## Integration with Unified Interface

See **14-unified-interface.md** for how these different root computation strategies fit into a single user-facing API:

```rust
// User code is IDENTICAL regardless of scheme
type Scheme = BalancedTreeScheme; // or MmrScheme

let mut db: Qmdb<Scheme> = Qmdb::new();
let sn = db.add(entry_hash)?;
let root = db.root();  // compute_root() called internally
let proof = db.prove(sn)?;
assert!(proof.verify(&entry_hash, &root)?);
```

The different root computation (MMR grafting vs balanced tree overlay) is encapsulated inside `StateRootScheme::compute_root()`, invisible to users.

## Prototype: Balanced Tree Root Computation

```rust
// Complete prototype showing the flow

pub fn compute_balanced_tree_root(
    segments: &[SegmentState],
    hasher: &mut LevelKeyed,
) -> Hash32 {
    // 1. Each segment already has segment_root computed:
    //    segment_root = H_11(entry_root, bitmap_root)

    // 2. Build upper tree from segment roots
    let mut builder = BalancedTreeRootBuilder::new(hasher.fork());

    for (id, segment) in segments.iter().enumerate() {
        // Segment root enters at level 12
        builder.add_segment(id as u64, segment.segment.segment_root);
    }

    // 3. Finalize: merge siblings, pad with nulls, return apex
    builder.finalize()
}

// Compare to MMR root computation:
pub async fn compute_mmr_root<H: CHasher>(
    status: &CleanBitMap<H::Digest, N>,
    mmr: &Mmr<E, H::Digest, Clean<DigestOf<H>>>,
    hasher: &mut StandardHasher<H>,
    height: u32,
) -> Result<H::Digest, Error> {
    // Uses existing grafting::Storage to combine bitmap and MMR
    let grafted = grafting::Storage::new(status, mmr, height);
    grafted.root(hasher).await
}
```

## Conclusion

The balanced tree scheme's tree combination is **not** grafting in the MMR sense. It's simpler fixed-level hashing that's already handled by:
1. `Segment::sync_root()` at level 11
2. `BalancedTreeRootBuilder` at levels 12+
3. `StateRootScheme::compute_root()` as the abstraction

No changes to `grafting.rs` are needed. The existing plan (Tasks 02-05) already handles everything correctly.
