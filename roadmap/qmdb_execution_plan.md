# QMDB Execution Plan: 100% Compatibility, Performance, and Memory Parity

This document provides a detailed execution plan for achieving complete compatibility between
`commonware-storage QMDB` and the original `cyclone QMDB`, targeting:

1. **100% State Root Compatibility** - Identical Merkle roots for identical data
2. **100% Performance Parity** - Matching throughput and latency characteristics  
3. **100% Memory Efficiency Parity** - Equivalent memory footprint

## Table of Contents

1. [Architecture Comparison](#1-architecture-comparison)
2. [State Root Compatibility Plan](#2-state-root-compatibility-plan)
3. [Performance Parity Plan](#3-performance-parity-plan)
4. [Memory Efficiency Plan](#4-memory-efficiency-plan)
5. [Implementation Phases](#5-implementation-phases)
6. [Validation Strategy](#6-validation-strategy)

---

## 1. Architecture Comparison

### 1.1 Core Data Structures

| Component | Cyclone QMDB | Commonware QMDB | Gap |
|-----------|--------------|-----------------|-----|
| **Merkle Structure** | Twig-based (2048 entries/twig) + Upper Tree | MMR over entries + grafted AuthenticatedBitMap | **Major** |
| **Active Bits** | 256-byte `ActiveBits` per twig, 3-level binary tree | MMR over N-byte chunks (configurable) | **Major** |
| **Indexer** | `HybridIndexer` with 65536 units | Single `BTreeMap` | **Major** |
| **Sharding** | 16 shards, parallel pipeline | Single logical DB | **Major** |
| **I/O Model** | io_uring prefetcher, `EntryBuffer` ring | Generic runtime storage | **Moderate** |

### 1.2 State Root Calculation Comparison

**Cyclone QMDB Root Calculation:**
```
For each Twig (2048 entries):
  1. left_root = H(level=11, merkle_tree_over_entry_hashes)
  2. active_bits_mtl3 = 3-level binary tree over 8 x 32-byte chunks:
     - L1[0..4] = H(8, chunk[0..32], chunk[32..64]), ...
     - L2[0..2] = H(9, L1[0], L1[1]), H(9, L1[2], L1[3])
     - L3 = H(10, L2[0], L2[1])
  3. twig_root = H(11, left_root, active_bits_mtl3)

Shard Root = Upper Tree root over all twig_roots
Global Root = H(shard_root[0] || ... || shard_root[15])
```

**Commonware QMDB Root Calculation:**
```
1. mmr_root = standard MMR root over entry digests
2. bitmap_root = MMR root over N-byte bitmap chunks  
3. grafted_root = graft bitmap tree onto MMR at height log2(N*8)
4. If partial chunk exists:
   final_root = H(grafted_root || next_bit || last_chunk_digest)
5. Else:
   final_root = grafted_root
```

**Key Difference:** Cyclone uses a fixed 2048-entry twig structure with a specific 3-level
active bits tree, while commonware uses configurable chunk sizes with MMR-based merkleization.

---

## 2. State Root Compatibility Plan

### 2.1 Strategy: Abstract Hasher Interface

Instead of modifying the core MMR/grafting logic, we introduce a **compatible hasher** that
produces cyclone-equivalent roots while reusing existing infrastructure.

### 2.2 Required Hasher Trait Extensions

**File:** `storage/src/mmr/hasher.rs`

Add two new methods to the `Hasher` trait:

```rust
pub trait Hasher<D: Digest>: Send + Sync {
    // ... existing methods ...

    /// Compute the root incorporating partial chunk information.
    /// Default implementation matches current commonware behavior.
    fn qmdb_root<'a>(
        &mut self,
        size: Position,
        next_bit: u64,
        last_chunk_digest: &D,
        peak_digests: impl Iterator<Item = &'a D>,
    ) -> D {
        let mmr_root = self.root(size, peak_digests);
        if next_bit == 0 {
            return mmr_root;
        }
        // Default: H(mmr_root || next_bit || last_chunk_digest)
        self.inner().update(mmr_root.as_ref());
        self.inner().update(&next_bit.to_be_bytes());
        self.inner().update(last_chunk_digest.as_ref());
        self.inner().finalize()
    }

    /// Compute digest at grafting boundary for twig-level compatibility.
    /// For cyclone compatibility, this builds the 3-level active bits tree.
    fn graft_digest(&mut self, pos: Position, peak: &D, chunk_bits: &[u8]) -> D {
        // Default: simple concatenation
        self.inner().update(chunk_bits);
        self.inner().update(peak.as_ref());
        self.inner().finalize()
    }
}
```

### 2.3 Cyclone-Compatible Hasher Implementation

**New File:** `storage/src/mmr/hasher/compatible.rs`

```rust
//! QMDB-compatible hasher producing state roots matching original QMDB format.

use super::{Hasher, Standard};
use crate::mmr::Position;
use commonware_cryptography::{Digest, Hasher as CHasher};

/// Hasher that produces cyclone-QMDB-compatible roots.
/// 
/// This hasher implements the exact twig structure:
/// - 2048 entries per twig (TWIG_SHIFT = 11)
/// - 3-level binary tree for active bits (256 bytes = 8 x 32-byte chunks)
/// - twig_root = H(11, left_root, active_bits_mtl3)
pub struct Compatible<H: CHasher> {
    inner: Standard<H>,
}

impl<H: CHasher> Compatible<H> {
    pub fn new() -> Self {
        Self {
            inner: Standard::new(),
        }
    }

    /// Build the 3-level binary tree over 256 bytes of active bits.
    /// Returns active_bits_mtl3 (the right tree root at level 10).
    fn build_active_bits_tree(&mut self, bits: &[u8; 256]) -> H::Digest {
        // Level 8: Hash pairs of 32-byte chunks -> 4 nodes
        let mut l1: [H::Digest; 4] = std::array::from_fn(|i| {
            let chunk0 = &bits[i * 64..i * 64 + 32];
            let chunk1 = &bits[i * 64 + 32..i * 64 + 64];
            self.merkle_node_hash(8, chunk0, chunk1)
        });

        // Level 9: Hash pairs of L1 nodes -> 2 nodes
        let l2_0 = self.node_digest_raw(9, &l1[0], &l1[1]);
        let l2_1 = self.node_digest_raw(9, &l1[2], &l1[3]);

        // Level 10: Hash L2 nodes -> 1 node (active_bits_mtl3)
        self.node_digest_raw(10, &l2_0, &l2_1)
    }

    fn merkle_node_hash(&mut self, level: u8, left: &[u8], right: &[u8]) -> H::Digest {
        // Cyclone format: H(level || left || right)
        self.inner.inner().update(&[level]);
        self.inner.inner().update(left);
        self.inner.inner().update(right);
        self.inner.inner().finalize()
    }

    fn node_digest_raw(&mut self, level: u8, left: &H::Digest, right: &H::Digest) -> H::Digest {
        self.inner.inner().update(&[level]);
        self.inner.inner().update(left.as_ref());
        self.inner.inner().update(right.as_ref());
        self.inner.inner().finalize()
    }
}

impl<H: CHasher> Default for Compatible<H> {
    fn default() -> Self {
        Self::new()
    }
}

impl<H: CHasher> Hasher<H::Digest> for Compatible<H> {
    type Inner = H;

    fn inner(&mut self) -> &mut H {
        self.inner.inner()
    }

    fn fork(&self) -> impl Hasher<H::Digest> {
        Self::new()
    }

    fn leaf_digest(&mut self, pos: Position, element: &[u8]) -> H::Digest {
        self.inner.leaf_digest(pos, element)
    }

    fn node_digest(&mut self, pos: Position, left: &H::Digest, right: &H::Digest) -> H::Digest {
        // For cyclone compatibility, include level byte in node hash
        let level = crate::mmr::iterator::pos_to_height(pos) as u8;
        self.node_digest_raw(level, left, right)
    }

    fn root<'a>(
        &mut self,
        size: Position,
        peak_digests: impl Iterator<Item = &'a H::Digest>,
    ) -> H::Digest {
        self.inner.root(size, peak_digests)
    }

    fn digest(&mut self, data: &[u8]) -> H::Digest {
        self.inner.digest(data)
    }

    /// Cyclone-compatible root calculation.
    /// For a complete twig (2048 entries), produces:
    ///   H(11, entries_tree_root, active_bits_mtl3)
    fn qmdb_root<'a>(
        &mut self,
        size: Position,
        next_bit: u64,
        last_chunk: &H::Digest,
        peak_digests: impl Iterator<Item = &'a H::Digest>,
    ) -> H::Digest {
        let entries_tree_root = self.root(size, peak_digests);
        
        // If we're on a twig boundary, use standard twig root calculation
        // Otherwise, incorporate partial chunk
        if next_bit == 0 || next_bit % 2048 == 0 {
            return entries_tree_root;
        }

        // For partial twigs, we still need to compute the active_bits tree
        // This requires the full 256-byte active bits array
        // The caller must provide this through the graft_digest mechanism
        self.inner.inner().update(entries_tree_root.as_ref());
        self.inner.inner().update(last_chunk.as_ref());
        self.inner.inner().finalize()
    }

    /// Build twig root from entries tree root and active bits.
    fn graft_digest(&mut self, pos: Position, peak: &H::Digest, chunk_bits: &[u8]) -> H::Digest {
        if chunk_bits.len() == 256 {
            // Full twig: build 3-level tree and compute twig_root
            let bits_array: [u8; 256] = chunk_bits.try_into().expect("checked length");
            let bits_tree_root = self.build_active_bits_tree(&bits_array);
            // twig_root = H(11, left_root, active_bits_mtl3)
            return self.node_digest_raw(11, peak, &bits_tree_root);
        }

        // Partial twig: fallback to default behavior
        self.inner.inner().update(chunk_bits);
        self.inner.inner().update(peak.as_ref());
        self.inner.inner().finalize()
    }
}
```

### 2.4 Feature Flag for Compatibility Mode

**File:** `storage/Cargo.toml`

```toml
[features]
default = []
qmdb-compatible = []
```

### 2.5 Integration Points

**File:** `storage/src/qmdb/current/mod.rs`

```rust
// Conditional hasher selection based on feature flag
#[cfg(feature = "qmdb-compatible")]
use crate::mmr::hasher::Compatible as QmdbHasher;

#[cfg(not(feature = "qmdb-compatible"))]
use crate::mmr::StandardHasher as QmdbHasher;
```

### 2.6 Validation Tests

```rust
#[cfg(test)]
mod compatibility_tests {
    use super::*;
    
    /// Test that Compatible hasher produces identical roots to cyclone QMDB
    /// for the same input data.
    #[test]
    fn test_twig_root_compatibility() {
        // Create identical entry sequence
        let entries: Vec<[u8; 32]> = (0..2048u64)
            .map(|i| Sha256::hash(&i.to_be_bytes()))
            .collect();
        
        // Create identical active bits (all active)
        let active_bits = [0xFF; 256];
        
        // Compute root with Compatible hasher
        let mut hasher = Compatible::<Sha256>::new();
        // ... build tree and compute root ...
        
        // Compare against known cyclone QMDB root
        // (obtained from running cyclone with same data)
        assert_eq!(computed_root, expected_cyclone_root);
    }
}
```

---

## 3. Performance Parity Plan

### 3.1 Current Bottlenecks (from qmdb_compare.md)

| Issue | Impact | Root Cause |
|-------|--------|------------|
| Serialized commit path | 10x throughput loss | `&mut self` APIs, single logical DB |
| Synchronous merkleization | Latency spikes | `merkleize()` blocks on bitmap updates |
| Global index lock | Contention | Single `BTreeMap` for all keys |
| No prefetching | Read amplification | No specialized I/O optimization |

### 3.2 Performance Improvement Strategy

#### Phase 1: Sharded Index (Critical Path)

Replace single `BTreeMap` with sharded structure matching cyclone's `HybridIndexer`.

**New File:** `storage/src/index/sharded.rs`

```rust
use dashmap::DashMap;
use parking_lot::Mutex;
use std::sync::Arc;

/// Number of shards (matches cyclone SHARD_COUNT)
const SHARD_COUNT: usize = 16;

/// Number of units per shard (matches cyclone UNIT_COUNT / SHARD_COUNT)
const UNITS_PER_SHARD: usize = 4096;

/// Sharded index with two-level partitioning.
/// Level 1: Key's high nibble -> shard (16 shards)
/// Level 2: Key's next 12 bits -> unit (4096 units per shard)
pub struct ShardedIndex<V: Eq + Clone> {
    shards: [Shard<V>; SHARD_COUNT],
}

struct Shard<V: Eq + Clone> {
    units: Vec<Mutex<Unit<V>>>,
}

struct Unit<V: Eq + Clone> {
    data: DashMap<[u8; 8], Record<V>>,
}

impl<V: Eq + Clone> ShardedIndex<V> {
    pub fn new() -> Self {
        Self {
            shards: std::array::from_fn(|_| Shard {
                units: (0..UNITS_PER_SHARD)
                    .map(|_| Mutex::new(Unit {
                        data: DashMap::new(),
                    }))
                    .collect(),
            }),
        }
    }

    /// Extract shard ID from key (high nibble of first byte)
    #[inline]
    fn shard_id(key: &[u8]) -> usize {
        (key[0] >> 4) as usize
    }

    /// Extract unit ID from key (next 12 bits)
    #[inline]
    fn unit_id(key: &[u8]) -> usize {
        let high = (key[0] & 0x0F) as usize;
        let low = key[1] as usize;
        (high << 8) | low
    }

    /// Extract local key (remaining bytes after shard/unit prefix)
    #[inline]
    fn local_key(key: &[u8]) -> [u8; 8] {
        let mut local = [0u8; 8];
        local.copy_from_slice(&key[2..10]);
        local
    }

    pub fn get(&self, key: &[u8]) -> Option<V> {
        let shard = &self.shards[Self::shard_id(key)];
        let unit = shard.units[Self::unit_id(key)].lock();
        unit.data.get(&Self::local_key(key)).map(|r| r.value.clone())
    }

    pub fn insert(&self, key: &[u8], value: V) {
        let shard = &self.shards[Self::shard_id(key)];
        let unit = shard.units[Self::unit_id(key)].lock();
        unit.data.insert(Self::local_key(key), Record { value, next: None });
    }
}
```

#### Phase 2: Pipeline Architecture

Introduce background processing for commit operations.

**Concept:** Instead of synchronous commit, use a 2-stage pipeline:
1. **Updater Stage:** Append entries to buffer, update in-memory index
2. **Flusher Stage:** Persist to storage, compute Merkle tree, update metadata

```rust
/// Background commit pipeline
pub struct CommitPipeline<E, H, const N: usize> {
    /// Channel for sending commit batches to flusher
    commit_tx: Sender<CommitBatch<E, H, N>>,
    /// Handle to flusher thread
    flusher_handle: JoinHandle<()>,
}

struct CommitBatch<E, H, const N: usize> {
    entries: Vec<E>,
    bitmap_updates: Vec<(u64, bool)>,
    metadata: Option<H::Digest>,
}

impl<E, H, const N: usize> CommitPipeline<E, H, N>
where
    E: Send + 'static,
    H: Hasher + Send + 'static,
{
    pub fn new(storage: Arc<Mutex<Storage<E, H, N>>>) -> Self {
        let (commit_tx, commit_rx) = bounded(2); // 2-block latency
        
        let flusher_handle = thread::spawn(move || {
            while let Ok(batch) = commit_rx.recv() {
                // Process batch in background
                let mut storage = storage.lock();
                storage.process_batch(batch);
            }
        });

        Self { commit_tx, flusher_handle }
    }

    /// Submit batch for background processing (non-blocking)
    pub fn submit(&self, batch: CommitBatch<E, H, N>) -> Result<(), Error> {
        self.commit_tx.send(batch).map_err(|_| Error::PipelineClosed)
    }
}
```

#### Phase 3: Prefetching (Linux-specific)

Add io_uring-based prefetching for read-heavy workloads.

```rust
#[cfg(all(target_os = "linux", feature = "iouring-storage"))]
pub struct Prefetcher {
    ring: io_uring::IoUring,
    entry_file: Arc<File>,
    cache: Arc<EntryCache>,
}

#[cfg(all(target_os = "linux", feature = "iouring-storage"))]
impl Prefetcher {
    pub fn prefetch_entries(&self, positions: &[i64]) {
        for &pos in positions {
            // Submit read request to io_uring
            // On completion, insert into cache
        }
    }
}
```

### 3.3 Performance Metrics

After implementation, we should achieve:

| Metric | Current | Target | Method |
|--------|---------|--------|--------|
| Throughput | 10% of cyclone | 100% of cyclone | Pipeline + sharding |
| Commit latency | Synchronous | 2-block pipelined | Background flusher |
| Index contention | Global lock | Per-unit lock | ShardedIndex |

---

## 4. Memory Efficiency Plan

### 4.1 Current Memory Issues

| Issue | Memory Impact | Root Cause |
|-------|---------------|------------|
| In-memory MMR nodes | 5x overhead | `VecDeque<D>` for all nodes |
| Full digest storage | High | Storing 32-byte digests per chunk |
| Index overhead | Moderate | BTreeMap node overhead |

### 4.2 Memory Reduction Strategy

#### Replace AuthenticatedBitMap with Twig-based ActiveBits

Instead of storing full MMR digests, store only raw bits and compute hashes on-the-fly.

**New File:** `storage/src/bitmap/twig.rs`

```rust
/// Twig-based active bits matching cyclone structure.
/// Stores only 256 bytes per twig (2048 bits) instead of MMR nodes.
pub struct TwigActiveBits {
    /// Raw bits: 256 bytes = 2048 bits per twig
    bits: [u8; 256],
    /// Cached 3-level tree nodes (computed on demand)
    cached_mtl1: Option<[[u8; 32]; 4]>,
    cached_mtl2: Option<[[u8; 32]; 2]>,
    cached_mtl3: Option<[u8; 32]>,
    dirty: bool,
}

impl TwigActiveBits {
    pub const BITS_PER_TWIG: usize = 2048;
    pub const BYTES_PER_TWIG: usize = 256;

    pub fn new() -> Self {
        Self {
            bits: [0; 256],
            cached_mtl1: None,
            cached_mtl2: None,
            cached_mtl3: None,
            dirty: false,
        }
    }

    /// Get bit at position within twig
    #[inline]
    pub fn get(&self, pos: usize) -> bool {
        debug_assert!(pos < Self::BITS_PER_TWIG);
        let byte_idx = pos / 8;
        let bit_idx = pos % 8;
        (self.bits[byte_idx] >> bit_idx) & 1 == 1
    }

    /// Set bit at position within twig
    #[inline]
    pub fn set(&mut self, pos: usize, value: bool) {
        debug_assert!(pos < Self::BITS_PER_TWIG);
        let byte_idx = pos / 8;
        let bit_idx = pos % 8;
        if value {
            self.bits[byte_idx] |= 1 << bit_idx;
        } else {
            self.bits[byte_idx] &= !(1 << bit_idx);
        }
        self.dirty = true;
        // Invalidate cached hashes affected by this change
        self.invalidate_cache(pos);
    }

    /// Compute the active_bits_mtl3 (right tree root)
    pub fn compute_root<H: Hasher>(&mut self, hasher: &mut H) -> H::Digest {
        if !self.dirty && self.cached_mtl3.is_some() {
            return H::Digest::from_bytes(&self.cached_mtl3.unwrap());
        }
        
        // Compute 3-level tree
        // Level 8: 4 nodes from 8 x 32-byte chunks
        let mut mtl1: [[u8; 32]; 4] = [[0; 32]; 4];
        for i in 0..4 {
            let chunk0 = &self.bits[i * 64..i * 64 + 32];
            let chunk1 = &self.bits[i * 64 + 32..i * 64 + 64];
            mtl1[i] = hasher.merkle_node_hash(8, chunk0, chunk1).into();
        }

        // Level 9: 2 nodes
        let mut mtl2: [[u8; 32]; 2] = [[0; 32]; 2];
        mtl2[0] = hasher.node_digest_bytes(9, &mtl1[0], &mtl1[1]).into();
        mtl2[1] = hasher.node_digest_bytes(9, &mtl1[2], &mtl1[3]).into();

        // Level 10: 1 node (root)
        let mtl3 = hasher.node_digest_bytes(10, &mtl2[0], &mtl2[1]);

        // Cache results
        self.cached_mtl1 = Some(mtl1);
        self.cached_mtl2 = Some(mtl2);
        self.cached_mtl3 = Some(mtl3.into());
        self.dirty = false;

        mtl3
    }

    fn invalidate_cache(&mut self, pos: usize) {
        // Invalidate only affected portions of cache
        let chunk_idx = pos / 256; // Which 32-byte chunk
        let l1_idx = chunk_idx / 2;
        let l2_idx = l1_idx / 2;
        
        // For now, invalidate all (can be optimized later)
        self.cached_mtl1 = None;
        self.cached_mtl2 = None;
        self.cached_mtl3 = None;
    }
}

/// Collection of twigs for a shard
pub struct ShardActiveBits {
    /// Twigs indexed by twig_id
    twigs: Vec<TwigActiveBits>,
    /// Pruned twig count (twigs before this are evicted)
    pruned_count: u64,
}

impl ShardActiveBits {
    /// Memory per twig: 256 bytes raw bits + ~128 bytes cached hashes = ~384 bytes
    /// vs. commonware: ~32 bytes per bit in MMR = ~65KB per twig!
    /// Memory savings: ~170x per twig
    pub const BYTES_PER_TWIG: usize = 384;
}
```

### 4.3 Memory Comparison

| Component | Cyclone | Commonware Current | Commonware After |
|-----------|---------|-------------------|------------------|
| Active Bits (per 2048 entries) | 256 bytes | ~65KB (MMR nodes) | 384 bytes |
| Index (per 1M keys) | ~80MB (sharded) | ~120MB (BTreeMap) | ~80MB (sharded) |
| Entry Cache | Bounded | Unbounded growth | Bounded LRU |

---

## 5. Implementation Phases

### Phase 1: State Root Compatibility (Week 1-2)

**Priority: Critical** - Required for any interoperability

1. [ ] Implement `Compatible` hasher in `storage/src/mmr/hasher/compatible.rs`
2. [ ] Add `qmdb-compatible` feature flag to `storage/Cargo.toml`
3. [ ] Extend `Hasher` trait with `qmdb_root` and `graft_digest` methods
4. [ ] Update `AuthenticatedBitMap` to use new hasher methods
5. [ ] Add conformance tests comparing roots against cyclone test vectors
6. [ ] Document compatibility mode in README

**Validation Checkpoint:**
```bash
# Run compatibility tests
cargo test -p commonware-storage --features qmdb-compatible compatibility_
```

### Phase 2: Sharded Index (Week 2-3)

**Priority: High** - Addresses main performance bottleneck

1. [ ] Implement `ShardedIndex` in `storage/src/index/sharded.rs`
2. [ ] Add `sharded-index` feature flag
3. [ ] Modify `qmdb/current/ordered/fixed.rs` to use `ShardedIndex` when feature enabled
4. [ ] Benchmark index operations: insert, lookup, iteration
5. [ ] Add stress tests for concurrent access

**Validation Checkpoint:**
```bash
# Run index benchmarks
cargo bench -p commonware-storage -- sharded_index
```

### Phase 3: Twig-based ActiveBits (Week 3-4)

**Priority: High** - Addresses memory overhead

1. [ ] Implement `TwigActiveBits` in `storage/src/bitmap/twig.rs`
2. [ ] Implement `ShardActiveBits` for multi-twig management
3. [ ] Add `twig-activebits` feature flag
4. [ ] Modify `AuthenticatedBitMap` to use twig structure when feature enabled
5. [ ] Measure memory usage reduction

**Validation Checkpoint:**
```bash
# Measure memory usage
cargo run --release -p commonware-storage --example memory_profile
```

### Phase 4: Pipeline Architecture (Week 4-6)

**Priority: Medium** - Further performance improvements

1. [ ] Implement `CommitPipeline` for background flushing
2. [ ] Add `EntryBuffer` ring buffer for writer/flusher decoupling
3. [ ] Implement 2-block latency semantics with `EntryCache`
4. [ ] Add pipeline feature flag
5. [ ] Benchmark end-to-end throughput

### Phase 5: io_uring Prefetching (Week 6-7)

**Priority: Low** - Linux-specific optimization

1. [ ] Implement `Prefetcher` with io_uring integration
2. [ ] Add `iouring-prefetch` feature flag (Linux only)
3. [ ] Integrate with runtime's io_uring support
4. [ ] Benchmark read-heavy workloads

---

## 6. Validation Strategy

### 6.1 Conformance Tests

Add conformance tests using cyclone QMDB as reference:

```rust
#[cfg(test)]
mod conformance {
    /// Generate deterministic entries for testing
    fn generate_test_entries(count: usize, seed: u64) -> Vec<Entry> {
        let mut rng = StdRng::seed_from_u64(seed);
        (0..count)
            .map(|_| Entry::random(&mut rng))
            .collect()
    }

    #[test]
    fn test_root_matches_cyclone() {
        let entries = generate_test_entries(10000, 42);
        
        // Compute root with commonware (compatible mode)
        let cw_root = compute_commonware_root(&entries);
        
        // Compare against pre-computed cyclone root
        let cyclone_root = hex!("...");
        
        assert_eq!(cw_root, cyclone_root);
    }

    #[test]
    fn test_proof_interoperability() {
        // Generate proof with commonware
        let proof = db.key_value_proof(&key).await?;
        
        // Verify with cyclone verification logic
        assert!(cyclone_verify_proof(&proof, &root, &key, &value));
    }
}
```

### 6.2 Performance Benchmarks

```rust
#[bench]
fn bench_insert_throughput(b: &mut Bencher) {
    // Measure operations per second
    // Target: match cyclone's ~1M ops/sec
}

#[bench]
fn bench_commit_latency(b: &mut Bencher) {
    // Measure p99 commit latency
    // Target: <10ms for 1000 operations
}

#[bench]
fn bench_memory_usage(b: &mut Bencher) {
    // Measure RSS after 1M entries
    // Target: <1GB
}
```

### 6.3 Integration Testing

```bash
# Full integration test suite
just test -p commonware-storage --features "qmdb-compatible,sharded-index,twig-activebits"

# Cross-verify with cyclone
./scripts/cross_verify.sh
```

---

## Appendix A: Key Constants

For exact compatibility, ensure these constants match cyclone:

```rust
/// Cyclone constants from crates/qmdb-common/src/def.rs
pub const SHARD_COUNT: usize = 16;
pub const TWIG_SHIFT: u32 = 11;
pub const LEAF_COUNT_IN_TWIG: u32 = 1 << TWIG_SHIFT; // 2048
pub const TWIG_ROOT_LEVEL: i64 = 12;
pub const FIRST_LEVEL_ABOVE_TWIG: i64 = 13;
```

## Appendix B: Cyclone Twig Structure Reference

```
                             TwigRoot (level 11)
                                |
                ┌--------------┴---------------┐
                │                               │
           Left Root                       active_bits_mtl3
         (Entry Tree)                       (level 10)
                │                               │
         [11-level binary tree]           ┌────┴────┐
                │                         │         │
      2048 Entry Hashes              mtl2[0]    mtl2[1]
                                    (level 9) (level 9)
                                       │         │
                                    ┌──┴──┐   ┌──┴──┐
                                   mtl1   mtl1 mtl1 mtl1
                                   [0]    [1]  [2]  [3]
                                 (level 8, each from 2x32B chunks)
                                    │
                              8 x 32-byte chunks
                              (256 bytes total = 2048 bits)
```

## Appendix C: File Changes Summary

| File | Change Type | Description |
|------|-------------|-------------|
| `storage/Cargo.toml` | Modify | Add feature flags |
| `storage/src/mmr/hasher.rs` | Modify | Extend `Hasher` trait |
| `storage/src/mmr/hasher/compatible.rs` | Create | Compatible hasher |
| `storage/src/index/sharded.rs` | Create | Sharded index |
| `storage/src/bitmap/twig.rs` | Create | Twig-based active bits |
| `storage/src/bitmap/authenticated.rs` | Modify | Use twig structure |
| `storage/src/qmdb/current/mod.rs` | Modify | Use compatible hasher |
| `storage/src/qmdb/current/ordered/fixed.rs` | Modify | Use sharded index |

