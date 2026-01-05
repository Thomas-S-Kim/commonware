# Balanced Binary Tree State Root Scheme

Add balanced binary tree state root variant to commonware-storage.

## Problem

Two ways to finalize state roots from an MMR-structured log:
1. **MMR scheme**: Peaks connected via direct bagging (current commonware)
2. **Balanced tree scheme**: Virtual balanced binary tree overlaid on fixed-size segments

Need commonware to support both schemes for interoperability with systems using the balanced tree approach.

### Scheme Differences

| Aspect | MMR Scheme | Balanced Tree Scheme |
|--------|------------|----------------------|
| Hash function | Position-based | Blake3 level-keyed |
| Peak finalization | Direct bagging | Virtual tree overlay |
| Entries per segment | Configurable | Configurable (power of 2, default 2048) |
| Active bits | Flat bitmap | Hierarchical (3-level tree) |
| Domain separation | Position in hash input | Level XOR'd into Blake3 IV |

### Segment Size Configuration

Segment size is configurable at instantiation time via `SegmentConfig`:
- `shift: u8` - Log2 of entries per segment (valid range: 8-20)
- Default QMDB: shift=11 (2048 entries per segment)
- Configuration affects all derived values:
  - `entries_per_segment() = 2^shift`
  - `entry_root_level() = shift` (entry tree produces roots at this level)
  - `segment_root_level() = shift + 1` (upper tree combines at this level)

## Design Principles

1. **Minimize interface changes**: Reuse existing traits, create new implementations
2. **Clean/generic/universal interfaces**: Abstractions should work for both schemes
3. **Minimize new interfaces**: Prefer abstracting existing interfaces over parallel implementations
4. **Test-driven**: Validate against test vectors. Exact match or failure.

## Architecture

### Interface Strategy

**Reuse existing `Hasher<D>` trait** - create new implementation, not new trait:
```rust
// Existing trait in storage/src/mmr/hasher.rs - NO CHANGES
pub trait Hasher<D: Digest>: Send + Sync {
    fn leaf_digest(&mut self, pos: Position, element: &[u8]) -> D;
    fn node_digest(&mut self, pos: Position, left: &D, right: &D) -> D;
    fn root<'a>(&mut self, size: Position, peak_digests: impl Iterator<Item = &'a D>) -> D;
    // ...
}

// NEW: Implementation for balanced tree (derives level from position)
impl Hasher<Hash32> for LevelKeyed { ... }
```

**Add minimal new traits** for structural differences only:
```rust
// NEW: Unified scheme configuration (handles root computation and proofs)
pub trait StateRootScheme: Send + Sync + Clone + 'static {
    type Digest: Digest + Clone + Default + Send + Sync;
    type Hasher: Hasher<Self::Digest> + Send + Sync;
    type ChunkState: ChunkState<Digest = Self::Digest> + Send + Sync;
    type ProofMaterial: Clone + Send + Sync + 'static;  // Scheme-specific proof data
    type Config: Clone + Send + Sync + 'static;  // Scheme configuration (e.g., SegmentConfig)

    fn entries_per_chunk(&self) -> u32;  // Runtime configurable
    fn new_hasher(&self) -> Self::Hasher;
    fn new_chunk(&self) -> Self::ChunkState;
    fn compute_root(&self, hasher: &mut Self::Hasher, chunks: &[Self::ChunkState]) -> Self::Digest;
    fn generate_proof(&self, hasher: &mut Self::Hasher, chunks: &[Self::ChunkState], serial_number: u64) -> Result<Self::ProofMaterial, SchemeError>;
    fn verify_proof(&self, hasher: &mut Self::Hasher, material: &Self::ProofMaterial, entry_hash: &Self::Digest, expected_root: &Self::Digest) -> Result<bool, SchemeError>;
    fn proof_is_active(material: &Self::ProofMaterial) -> bool;
}

// NEW: Chunk/segment abstraction (uses associated type)
pub trait ChunkState: Send + Sync + Default + Clone + 'static {
    type Digest: Digest + Clone;

    fn set_active(&mut self, index: u32) -> Result<(), SchemeError>;
    fn set_inactive(&mut self, index: u32) -> Result<(), SchemeError>;
    fn is_active(&self, index: u32) -> Result<bool, SchemeError>;
    fn set_entry_hash(&mut self, index: u32, hash: Self::Digest) -> Result<(), SchemeError>;
    fn compute_root<H: Hasher<Self::Digest>>(&mut self, hasher: &mut H) -> Result<Self::Digest, SchemeError>;
}

// Unified proof type - same API for all schemes
pub struct Proof<S: StateRootScheme> {
    pub serial_number: u64,
    material: S::ProofMaterial,
}

impl<S: StateRootScheme> Proof<S> {
    pub fn verify(&self, entry_hash: &S::Digest, expected_root: &S::Digest) -> Result<bool, SchemeError>;
    pub fn is_active(&self) -> bool;
}
```

### Interface Summary

| Interface | Change Type | Description |
|-----------|-------------|-------------|
| `Hasher<D>` | **No change** | Existing trait for node hashing |
| `Digest` | **No change** | Existing trait for hash outputs |
| `BitMap<D,N,S>` | **No change** | Existing MMR bitmap |
| `LevelKeyed` | **New impl** | Implements existing `Hasher<D>` |
| `StateRootScheme` | **New trait** | Unified scheme configuration + proofs |
| `ChunkState<D>` | **New trait** | Chunk/segment abstraction |
| `Proof<S>` | **New struct** | Unified proof type (wraps scheme-specific material) |
| `Qmdb<S>` | **New/Update** | Database interface generic over scheme |

**Total new traits: 2** (StateRootScheme, ChunkState)
**Total new structs: 1** (Proof<S>)
**Total unchanged traits: 3** (Hasher, Digest, BitMap-related)

### Component Hierarchy

```
StateRootScheme (trait)
├── MmrScheme
│   ├── Hasher: Standard<H>     (existing impl)
│   └── ChunkState: MmrChunkState (wrapper around existing BitMap)
└── BalancedTreeScheme
    ├── Hasher: LevelKeyed      (new impl of existing trait)
    └── ChunkState: SegmentState
        ├── Segment             (internal, not a trait)
        ├── HierarchicalBitmap  (internal, not a trait)
        └── BalancedTreeRootBuilder (internal, not a trait)
```

## Tasks

### Milestone 1: Core Hashing Behavior
Validates that leaf hashing (BLAKE2b) and node hashing (Blake3 level-keyed) match the reference implementation. Entry tree only, no bitmap.

**CRITICAL REQUIREMENT**: Milestone 1 MUST produce byte-for-byte identical merkle roots as the QMDB reference implementation for ALL edge cases including:
- Empty trees (all null entries)
- Single entry at various positions
- Full segments (2048 entries)
- Partial segments with null padding
- Multiple segments with upper tree combination
- Edge cases at segment boundaries

The `Hasher::root()` method must transparently produce QMDB-style roots when using `LevelKeyed`, enabling seamless switching between MMR and QMDB schemes via a single type parameter change.

```
[ ] 01a-test-vectors-core.md         - Test vectors: leaf hash, node hash, entry tree root
[ ] 02-level-keyed-hasher.md         - LevelKeyed impl (BLAKE2b leaves, Blake3 keyed nodes)
[ ] 03a-entry-tree.md                - Entry tree structure (configurable size via SegmentConfig)
[ ] 04-balanced-tree-root.md         - BalancedTreeRootBuilder (upper tree, level shift+1 and above)
[ ] 05a-core-hashing-tests.md        - Validate core hashing matches reference EXACTLY
```

**Milestone 1 Success**: Entry tree roots and state roots match reference implementation exactly (byte-for-byte).

### Milestone 2: ActiveBits Hashing
Adds hierarchical bitmap (3-level tree) and full segment structure.

```
[ ] 01b-test-vectors-bitmap.md       - Test vectors: bitmap tree, segment root, full state root
[ ] 03b-hierarchical-bitmap.md       - HierarchicalBitmap (8 pages, levels 8-10)
[ ] 03c-segment-structure.md         - Full Segment (entry tree + bitmap combined at level 11)
[ ] 05b-scheme-trait.md              - StateRootScheme + ChunkState + Proof (unified interface)
[ ] 06-compatibility-tests.md        - Full compatibility tests with bitmap
[ ] 07-documentation-feature-flag.md - Documentation and feature flag
```

**Milestone 2 Success**: Full state roots (with active bits) match reference implementation exactly.

### Phase 3: Performance
```
[ ] 08-define-workloads.md           - Benchmark workloads
[ ] 09-benchmark-cli.md              - Benchmark CLI
[ ] 10-storage-telemetry.md          - Storage telemetry
[ ] 11-observability-stack.md        - Observability stack
[ ] 12-ci-integration.md             - CI integration
```

### Analysis Documents
```
[x] 00-architecture-analysis.md      - Interface design analysis
[x] 13-grafting-analysis.md          - Grafting vs balanced tree combination
[x] 14-unified-interface.md          - Unified interface design (single API for both schemes)
```

### Dependencies

```
Milestone 1 (Core Hashing):
01a -> 02 -> 03a -> 04 -> 05a

Milestone 2 (ActiveBits):
01b -> 03b -> 03c -> 05b -> 06 -> 07
       (requires Milestone 1 complete)

Phase 3 (Performance):
08 -> 09 -> 11 -> 12
10 ----^
```

## References

### Balanced Tree Scheme (Reference Implementation)
- `qmdb-common/src/merkletree/hash.rs` - Level-keyed hash
- `qmdb-common/src/merkletree/twig.rs` - Segment structure
- `qmdb-common/src/utils/hasher.rs` - Blake3 keyed hash
- `qmdb-common/src/def.rs` - Constants

### Commonware Files
- `storage/src/mmr/hasher.rs` - Hasher<D> trait (reused, no changes)
- `storage/src/mmr/grafting.rs` - Grafting (no changes)
- `storage/src/bitmap/authenticated.rs` - BitMap (no changes)
- `storage/src/qmdb/current/mod.rs` - CurrentDb (add type parameter)

### Architecture Analysis
- `00-architecture-analysis.md` - Detailed interface analysis

## Success Criteria

### Phase 1: Compatibility
- [ ] `LevelKeyed` implements existing `Hasher<D>` trait with functional `root()` method
- [ ] Only 2 new traits: `StateRootScheme`, `ChunkState<D>`
- [ ] Balanced tree scheme produces **byte-for-byte identical** roots to QMDB reference
- [ ] Segment size configurable at instantiation via `SegmentConfig`
- [ ] Single interface: `Qmdb<S>` works identically for both `MmrScheme` and `BalancedTreeScheme`
- [ ] Existing tests pass unchanged
- [ ] Migration path documented

### Phase 2: Performance
- [ ] CI benchmarks with regression detection
- [ ] Both schemes benchmarked

### Long-term
- [ ] Default swappable to MMR scheme
- [ ] Reference implementation deprecated
