# Architecture Analysis: Minimizing Interface Changes

## Existing Interfaces

### 1. `Hasher<D>` trait (mmr/hasher.rs)

```rust
pub trait Hasher<D: Digest>: Send + Sync {
    fn leaf_digest(&mut self, pos: Position, element: &[u8]) -> D;
    fn node_digest(&mut self, pos: Position, left: &D, right: &D) -> D;
    fn root<'a>(&mut self, size: Position, peak_digests: impl Iterator<Item = &'a D>) -> D;
    fn digest(&mut self, data: &[u8]) -> D;
    fn inner(&mut self) -> &mut Self::Inner;
    fn fork(&self) -> impl Hasher<D>;
}
```

**Current implementation**: `Standard<H>` uses position-based domain separation.

### 2. `BitMap<D, N, S>` (bitmap/authenticated.rs)

- Uses MMR structure for chunk authentication
- Chunks of N bytes, hashed as MMR leaves
- Grafted onto operation MMR for proof efficiency

### 3. QMDB root calculation (qmdb/current/mod.rs)

```rust
async fn root<E, H, const N: usize>(
    hasher: &mut StandardHasher<H>,
    height: u32,
    status: &CleanBitMap<H::Digest, N>,
    mmr: &Mmr<E, H::Digest, Clean<DigestOf<H>>>,
) -> Result<H::Digest, Error>
```

Uses grafting to combine bitmap and operation MMR.

## Scheme Differences

| Aspect | MMR Scheme | Balanced Tree Scheme |
|--------|------------|----------------------|
| Domain separation | Position-based | Level-based |
| Chunk size | Variable (MMR peaks) | Fixed (2048 entries) |
| Root finalization | Peak bagging: `H(size \|\| peak1 \|\| ...)` | Virtual tree overlay |
| Bitmap structure | MMR-structured chunks | Hierarchical (3-level tree) |

## Analysis: What Can Be Reused

### Reusable with no changes:
- `Digest` trait and implementations
- Clean/Dirty state pattern
- Basic proof verification flow
- Storage abstractions

### Reusable with parameterization:
- `Hasher<D>` trait - **create new implementation**, don't modify trait
- Chunk/segment concept - generalize via new abstraction

### Must be new:
- Level-keyed hash function (Blake3 with level XOR'd into IV)
- Fixed-size segment structure with hierarchical bitmap
- Virtual tree root builder

## Recommended Approach

### Option A: Minimal - New Hasher Implementation Only

Create `LevelKeyed<H>` implementing existing `Hasher<D>` trait:

```rust
impl<H: CHasher> Hasher<H::Digest> for LevelKeyed<H> {
    fn leaf_digest(&mut self, pos: Position, element: &[u8]) -> H::Digest {
        // Level derived from position
        let level = pos_to_height(pos);
        blake3_level_keyed(level, element)
    }

    fn node_digest(&mut self, pos: Position, left: &D, right: &D) -> H::Digest {
        let level = pos_to_height(pos);
        blake3_level_keyed(level, &[left, right])
    }

    fn root<'a>(&mut self, size: Position, peaks: impl Iterator<Item = &'a D>) -> H::Digest {
        // Build virtual balanced tree from peaks
        build_virtual_tree(peaks)
    }
}
```

**Problem**: The `root()` signature takes peaks, but balanced tree doesn't use peaks - it uses fixed segments. The semantics don't match.

### Option B: New Finalization Trait

Add a `RootFinalization` trait to separate root calculation:

```rust
pub trait RootFinalization<D: Digest>: Send + Sync {
    type Input;
    fn finalize(&mut self, input: Self::Input) -> D;
}

// MMR: Input = (size, peak_iterator)
// Balanced: Input = segment_iterator
```

**Problem**: Adds complexity, and doesn't solve the segment vs MMR structure issue.

### Option C: Scheme Abstraction (Final Design)

The `StateRootScheme` trait handles scheme configuration and root computation:

```rust
pub trait StateRootScheme: Send + Sync + Clone + 'static {
    type Digest: Digest + Clone + Default;
    type Hasher: Hasher<Self::Digest>;
    type ChunkState: ChunkState<Digest = Self::Digest>;

    const ENTRIES_PER_CHUNK: u32;

    fn new_hasher() -> Self::Hasher;
    fn new_chunk() -> Self::ChunkState;
    // Root computation at scheme level (MMR peak bagging vs balanced tree overlay)
    fn compute_root(
        hasher: &mut Self::Hasher,
        chunk_roots: impl Iterator<Item = (u64, Self::Digest)>,
    ) -> Self::Digest;
}

// ChunkState uses associated type (not generic parameter) per codebase convention
pub trait ChunkState: Send + Sync + Default + Clone + 'static {
    type Digest: Digest + Clone;

    fn set_active(&mut self, index: u32) -> Result<(), SchemeError>;
    fn set_inactive(&mut self, index: u32) -> Result<(), SchemeError>;
    fn is_active(&self, index: u32) -> Result<bool, SchemeError>;
    fn set_entry_hash(&mut self, index: u32, hash: Self::Digest) -> Result<(), SchemeError>;
    fn compute_root<H: Hasher<Self::Digest>>(&mut self, hasher: &mut H) -> Result<Self::Digest, SchemeError>;
}
```

Key design decisions:
1. **Associated type for Digest** - per codebase convention (e.g., `storage/src/mmr/mem.rs`)
2. **Root computation at scheme level** - avoids TypeId dispatch anti-pattern
3. **Result return types** - adversarial safety (no `debug_assert!` or panics)
4. **Clone + 'static bounds** - required for async and storage patterns

This approach:
1. **Reuses `Hasher<D>` trait** - new implementations, same interface
2. **Abstracts chunk management** - segments vs MMR chunks
3. **Minimal new traits** - just `StateRootScheme` + `ChunkState`

## Final Recommendation

**Use Option C (implemented):**

1. **`Hasher<D>` trait unchanged** - create `LevelKeyed` implementation
2. **Add `StateRootScheme` trait** - handles scheme configuration AND root computation
3. **Add `ChunkState` trait** - uses associated type (not generic parameter)
4. **No changes to existing `BitMap`** - balanced tree uses different structure entirely
5. **Parameterize QMDB** - `Db<..., S: StateRootScheme>`

## Interface Changes Summary

| Component | Change Type | Description |
|-----------|-------------|-------------|
| `Hasher<D>` | No change | Existing trait |
| `Digest` | No change | Existing trait |
| `Standard<H>` | No change | Existing implementation |
| `BitMap<D,N,S>` | No change | Used by MMR scheme only |
| `LevelKeyed` | New impl | Implements existing `Hasher<D>` at `storage/src/hasher/level_keyed.rs` |
| `Hash32` | New type | Digest impl for balanced tree |
| `StateRootScheme` | New trait | Scheme configuration + root computation |
| `ChunkState` | New trait | Associated type `Digest`, Result return types |
| `SchemeError` | New error | Error type for scheme operations |
| `Segment` | New struct | Internal to SegmentState (`pub(crate)`) |
| `HierarchicalBitmap` | New struct | Internal to SegmentState (`pub(crate)`) |
| `BitmapError` | New error | Error type for bitmap operations |
| `BalancedTreeRootBuilder` | New struct | Internal to BalancedTreeScheme |
| `SegmentState` | New struct | `pub(crate)`, implements `ChunkState` |
| QMDB `CurrentDb<...>` | Add param | Add `S: StateRootScheme` |

**Summary:**
- Unchanged traits: 2 (`Hasher<D>`, `Digest`)
- New traits: 2 (`StateRootScheme`, `ChunkState`)
- New error types: 2 (`SchemeError`, `BitmapError`)
- New implementations of existing traits: 1 (`LevelKeyed` impl `Hasher<D>`)
- New internal structs: 4 (`Segment`, `HierarchicalBitmap`, `BalancedTreeRootBuilder`, `SegmentState`)

## Module Organization

```
storage/src/
├── hasher/
│   ├── mod.rs                    # #[cfg(feature = "balanced_tree")]
│   └── level_keyed.rs            # LevelKeyed, Hash32
├── bitmap/
│   ├── mod.rs                    # existing + hierarchical export
│   ├── authenticated.rs          # existing (no change)
│   └── hierarchical.rs           # HierarchicalBitmap, Segment, BitmapError
├── qmdb/
│   ├── mod.rs                    # existing + scheme export
│   ├── current/
│   │   └── mod.rs                # CurrentDb<S: StateRootScheme>
│   ├── scheme/
│   │   ├── mod.rs                # StateRootScheme, ChunkState, SchemeError
│   │   ├── balanced_tree.rs      # BalancedTreeScheme, SegmentState
│   │   └── mmr.rs                # MmrScheme, MmrChunkState
│   └── balanced_tree_root.rs     # BalancedTreeRootBuilder (internal)
└── lib.rs                        # exports
```

## Key Principles

1. **Reuse existing traits** - `Hasher<D>` is the core interface for node hashing
2. **Associated types over generics** - per codebase convention
3. **Result return types** - adversarial safety, no panics on invalid input
4. **Scheme-level root computation** - avoids TypeId dispatch
5. **Internal structs are pub(crate)** - implementation details, not public API
6. **Feature flag: `balanced_tree`** - underscore, not hyphen
