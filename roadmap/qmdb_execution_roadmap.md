# QMDB 100% Compatibility & Performance Execution Roadmap

This document provides a comprehensive execution plan for achieving **100% performance and 100% memory parity** with the original QMDB implementation from [cyclone](https://github.com/LayerZero-Research/cyclone).

## Executive Summary

The current `commonware-storage` QMDB implementation uses a simplified MMR-based design that prioritizes correctness over performance. To achieve 100% parity, we must adopt the original QMDB's architecture:

| Component | Current (commonware) | Original (cyclone) | Action Required |
|-----------|---------------------|-------------------|-----------------|
| Authenticated Structure | Log MMR + AuthenticatedBitMap | Twig-based Merkle Tree | **Replace** |
| Chunk Size | 256 bits/chunk | 2048 bits/twig | **Change** |
| Right Tree | MMR of chunk digests | 3-level binary tree (8 leaves) | **Replace** |
| Indexer | Single BTreeMap | HybridIndexer (65536 units) | **Replace** |
| Concurrency | Serialized mutation | 16-shard parallel + pipeline | **Add** |
| Compaction | Synchronous `raise_floor` | Background Compactor threads | **Replace** |
| I/O | Generic runtime storage | io_uring + Prefetcher | **Add** |

---

## Part 1: Core Data Structure Changes

### 1.1 Twig-Based Merkle Tree (Critical)

The original QMDB uses a **Twig** as the fundamental unit, containing exactly 2048 entries:

```
             ____TwigRoot___                   Level_12
            /               \
           /                 \
        leftRoot              activeBitsMTL3   Level_11
        Level_10        2     activeBitsMTL2
        Level_9         4     activeBitsMTL1
        Level_8    8*32bytes  activeBits (256 bytes = 2048 bits)
        Level_7
        Level_6
        Level_5
        Level_4
        Level_3
        Level_2
        Level_1
        Level_0 (2048 entry hashes)
```

#### Step 1: Create Twig Structure

Create `storage/src/qmdb/twig.rs`:

```rust
//! Twig implementation matching original QMDB.
//!
//! A Twig manages exactly 2048 entries with:
//! - Left Tree: 11-level binary tree of entry hashes
//! - Right Tree: 3-level binary tree of active bits

use commonware_cryptography::{Digest, Hasher as CHasher};

/// Number of entries per twig (2^11 = 2048)
pub const TWIG_SHIFT: u32 = 11;
pub const LEAF_COUNT_IN_TWIG: u32 = 1 << TWIG_SHIFT; // 2048
pub const TWIG_MASK: u32 = LEAF_COUNT_IN_TWIG - 1;

/// Active bits for a single twig (256 bytes = 2048 bits)
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct ActiveBits(pub [u8; 256]);

impl Default for ActiveBits {
    fn default() -> Self {
        Self([0; 256])
    }
}

impl ActiveBits {
    /// Sets a bit to active at the specified offset (0-2047)
    #[inline]
    pub fn set_bit(&mut self, offset: u32) {
        debug_assert!(offset < LEAF_COUNT_IN_TWIG);
        let byte_idx = (offset / 8) as usize;
        let bit_idx = offset % 8;
        self.0[byte_idx] |= 1 << bit_idx;
    }

    /// Clears a bit at the specified offset (0-2047)
    #[inline]
    pub fn clear_bit(&mut self, offset: u32) {
        debug_assert!(offset < LEAF_COUNT_IN_TWIG);
        let byte_idx = (offset / 8) as usize;
        let bit_idx = offset % 8;
        self.0[byte_idx] &= !(1 << bit_idx);
    }

    /// Gets the bit at the specified offset
    #[inline]
    pub fn get_bit(&self, offset: u32) -> bool {
        debug_assert!(offset < LEAF_COUNT_IN_TWIG);
        let byte_idx = (offset / 8) as usize;
        let bit_idx = offset % 8;
        (self.0[byte_idx] >> bit_idx) & 1 != 0
    }

    /// Gets a 32-byte slice for the specified page (0-7)
    #[inline]
    pub fn get_page(&self, page: usize) -> &[u8; 32] {
        debug_assert!(page < 8);
        self.0[page * 32..(page + 1) * 32].try_into().unwrap()
    }

    /// Returns true if all bits are zero (twig can be evicted)
    pub fn is_empty(&self) -> bool {
        self.0.iter().all(|&b| b == 0)
    }
}

/// Merkle tree levels for the right tree (active bits)
#[derive(Clone, Debug, PartialEq)]
pub struct Twig<D: Digest> {
    /// Level 1 (8): 4 hashes, each from 2 pages of 32 bytes
    pub active_bits_mtl1: [D; 4],
    /// Level 2 (9): 2 hashes
    pub active_bits_mtl2: [D; 2],
    /// Level 3 (10): 1 hash (right root)
    pub active_bits_mtl3: D,
    /// Root hash of the left subtree (entry hashes)
    pub left_root: D,
    /// Root hash of the entire twig: H(left_root || active_bits_mtl3)
    pub twig_root: D,
}

impl<D: Digest> Default for Twig<D> {
    fn default() -> Self {
        Self {
            active_bits_mtl1: std::array::from_fn(|_| D::default()),
            active_bits_mtl2: std::array::from_fn(|_| D::default()),
            active_bits_mtl3: D::default(),
            left_root: D::default(),
            twig_root: D::default(),
        }
    }
}

impl<D: Digest> Twig<D> {
    /// Syncs level 1 of the active bits tree (8 -> 4 nodes)
    /// 
    /// Each L1 node = H(level=8 || page_n || page_n+1)
    pub fn sync_l1<H: CHasher<Digest = D>>(&mut self, hasher: &mut H, pos: usize, bits: &ActiveBits) {
        debug_assert!(pos < 4);
        let left = bits.get_page(pos * 2);
        let right = bits.get_page(pos * 2 + 1);
        
        hasher.update(&[8u8]); // level marker
        hasher.update(left);
        hasher.update(right);
        self.active_bits_mtl1[pos] = hasher.finalize();
    }

    /// Syncs level 2 of the active bits tree (4 -> 2 nodes)
    pub fn sync_l2<H: CHasher<Digest = D>>(&mut self, hasher: &mut H, pos: usize) {
        debug_assert!(pos < 2);
        hasher.update(&[9u8]); // level marker
        hasher.update(self.active_bits_mtl1[pos * 2].as_ref());
        hasher.update(self.active_bits_mtl1[pos * 2 + 1].as_ref());
        self.active_bits_mtl2[pos] = hasher.finalize();
    }

    /// Syncs level 3 of the active bits tree (2 -> 1 node)
    pub fn sync_l3<H: CHasher<Digest = D>>(&mut self, hasher: &mut H) {
        hasher.update(&[10u8]); // level marker
        hasher.update(self.active_bits_mtl2[0].as_ref());
        hasher.update(self.active_bits_mtl2[1].as_ref());
        self.active_bits_mtl3 = hasher.finalize();
    }

    /// Syncs the twig root: H(level=11 || left_root || active_bits_mtl3)
    pub fn sync_top<H: CHasher<Digest = D>>(&mut self, hasher: &mut H) {
        hasher.update(&[11u8]); // level marker
        hasher.update(self.left_root.as_ref());
        hasher.update(self.active_bits_mtl3.as_ref());
        self.twig_root = hasher.finalize();
    }

    /// Full sync of the active bits tree for a given set of touched positions
    pub fn sync_active_bits<H: CHasher<Digest = D>>(
        &mut self,
        hasher: &mut H,
        bits: &ActiveBits,
        touched_l1: &[usize], // which L1 positions need sync (0-3)
    ) {
        // Sync L1
        for &pos in touched_l1 {
            self.sync_l1(hasher, pos, bits);
        }
        
        // Sync L2 based on which L1 positions changed
        let mut l2_touched = [false; 2];
        for &pos in touched_l1 {
            l2_touched[pos / 2] = true;
        }
        for (pos, &touched) in l2_touched.iter().enumerate() {
            if touched {
                self.sync_l2(hasher, pos);
            }
        }
        
        // Sync L3
        if l2_touched[0] || l2_touched[1] {
            self.sync_l3(hasher);
        }
    }
}
```

#### Step 2: Create Upper Tree Structure

Create `storage/src/qmdb/upper_tree.rs`:

```rust
//! Upper tree management for levels above twigs (level 13+).

use commonware_cryptography::Digest;
use std::collections::HashMap;

/// Position in the Merkle tree encoded as (level, nth)
#[derive(Copy, Clone, Eq, Hash, PartialEq, Debug)]
pub struct NodePos {
    level: u8,
    nth: u64,
}

impl NodePos {
    pub fn new(level: u8, nth: u64) -> Self {
        Self { level, nth }
    }

    pub fn level(&self) -> u8 {
        self.level
    }

    pub fn nth(&self) -> u64 {
        self.nth
    }

    pub fn parent(&self) -> Self {
        Self {
            level: self.level + 1,
            nth: self.nth / 2,
        }
    }

    pub fn sibling(&self) -> Self {
        Self {
            level: self.level,
            nth: self.nth ^ 1,
        }
    }
}

/// Edge node for pruning - stores the boundary of pruned subtrees
#[derive(Debug, Clone)]
pub struct EdgeNode<D: Digest> {
    pub pos: NodePos,
    pub value: D,
}

/// Manages the upper levels of the Merkle tree (above twig level)
pub struct UpperTree<D: Digest> {
    /// Level 12 = twig roots, stored separately
    twig_roots: HashMap<u64, D>,
    /// Levels 13+ = internal nodes
    nodes: HashMap<NodePos, D>,
    /// ID of the youngest (most recent) twig
    youngest_twig_id: u64,
}

impl<D: Digest> UpperTree<D> {
    pub fn new() -> Self {
        Self {
            twig_roots: HashMap::new(),
            nodes: HashMap::new(),
            youngest_twig_id: 0,
        }
    }

    pub fn set_twig_root(&mut self, twig_id: u64, root: D) {
        self.twig_roots.insert(twig_id, root);
        if twig_id > self.youngest_twig_id {
            self.youngest_twig_id = twig_id;
        }
    }

    pub fn get_twig_root(&self, twig_id: u64) -> Option<&D> {
        self.twig_roots.get(&twig_id)
    }

    pub fn set_node(&mut self, pos: NodePos, value: D) {
        self.nodes.insert(pos, value);
    }

    pub fn get_node(&self, pos: NodePos) -> Option<&D> {
        self.nodes.get(&pos)
    }

    /// Calculate max level needed for current tree size
    pub fn calc_max_level(&self) -> u8 {
        if self.youngest_twig_id == 0 {
            return 12;
        }
        // bit_length = 64 - leading_zeros
        // max_level = 12 + bit_length
        12 + (64 - self.youngest_twig_id.leading_zeros()) as u8
    }

    /// Get edge nodes for pruning (stores boundary of pruned section)
    pub fn get_edge_nodes(&self, end_twig_id: u64) -> Vec<EdgeNode<D>> {
        let max_level = self.calc_max_level();
        let mut edges = Vec::new();
        let mut cur_end = end_twig_id;
        
        for level in 12..=max_level {
            let pos = NodePos::new(level, cur_end);
            if let Some(value) = if level == 12 {
                self.get_twig_root(cur_end)
            } else {
                self.get_node(pos)
            } {
                edges.push(EdgeNode { pos, value: value.clone() });
            }
            cur_end /= 2;
        }
        edges
    }
}
```

### 1.2 Replace AuthenticatedBitMap with Twig-based Structure

The current `AuthenticatedBitMap` uses an MMR over 256-bit chunks. We need to replace this with the twig-based structure that uses 2048 bits with a 3-level binary tree.

**Files to modify:**
- `storage/src/bitmap/authenticated.rs` - Replace with twig-based implementation
- `storage/src/qmdb/current/mod.rs` - Update root calculation
- `storage/src/qmdb/current/ordered/fixed.rs` - Update commit logic

#### Key Changes to `qmdb/current/mod.rs`:

```rust
// OLD: Using grafted MMR for root calculation
pub fn root<D: Digest>(
    hasher: &mut impl Hasher<D>,
    grafted_mmr: &mmr::Log<D>,
    status: &CleanBitMap<D, { CleanBitMap::<D, 256>::CHUNK_SIZE_BITS }>,
) -> D {
    // Complex grafting logic...
}

// NEW: Direct twig root aggregation
pub fn root<D: Digest, H: CHasher<Digest = D>>(
    hasher: &mut H,
    upper_tree: &UpperTree<D>,
) -> D {
    let max_level = upper_tree.calc_max_level();
    upper_tree.get_node(NodePos::new(max_level, 0))
        .cloned()
        .unwrap_or_default()
}
```

---

## Part 2: Sharding Architecture

### 2.1 16-Shard Parallel Design

The original QMDB uses 16 independent shards, each with its own:
- Entry file
- Merkle tree
- Indexer partition
- Background threads

```rust
//! Shard configuration matching original QMDB

pub const SHARD_COUNT: usize = 16;
pub const SHARD_SHIFT: u32 = 4; // log2(16)

/// Determines which shard a key belongs to based on first byte
#[inline]
pub fn key_to_shard(key_hash: &[u8]) -> usize {
    (key_hash[0] >> 4) as usize // High nibble (4 bits) = shard ID
}
```

### 2.2 Per-Shard Structure

Create `storage/src/qmdb/shard.rs`:

```rust
//! Per-shard state management

use super::twig::{ActiveBits, Twig, TWIG_SHIFT};
use super::upper_tree::UpperTree;
use commonware_cryptography::Digest;
use std::collections::HashMap;

/// State for a single QMDB shard
pub struct Shard<D: Digest> {
    /// Shard ID (0-15)
    pub id: usize,
    
    /// Upper tree containing twig roots and internal nodes
    pub upper_tree: UpperTree<D>,
    
    /// Active bits for each twig (keyed by twig_id)
    pub active_bits: HashMap<u64, ActiveBits>,
    
    /// Twig metadata for active twigs (right tree nodes)
    pub twigs: HashMap<u64, Twig<D>>,
    
    /// Merkle tree for the youngest twig's left tree (entry hashes)
    /// Size: 4096 elements (2048 leaves + 2047 internal + 1 root)
    pub youngest_twig_mt: Vec<D>,
    
    /// ID of the youngest twig
    pub youngest_twig_id: u64,
    
    /// Range of changes in youngest twig during current block
    pub mt_change_start: i32,
    pub mt_change_end: i32,
    
    /// Positions touched during current block (for batch sync)
    pub touched_positions: std::collections::HashSet<u64>,
}

impl<D: Digest> Shard<D> {
    pub fn new(id: usize) -> Self {
        Self {
            id,
            upper_tree: UpperTree::new(),
            active_bits: HashMap::new(),
            twigs: HashMap::new(),
            youngest_twig_mt: vec![D::default(); 4096],
            youngest_twig_id: 0,
            mt_change_start: -1,
            mt_change_end: -1,
            touched_positions: std::collections::HashSet::new(),
        }
    }

    /// Get active bit for a serial number
    pub fn get_active_bit(&self, sn: u64) -> bool {
        let twig_id = sn >> TWIG_SHIFT;
        let offset = (sn & ((1 << TWIG_SHIFT) - 1)) as u32;
        self.active_bits
            .get(&twig_id)
            .map(|bits| bits.get_bit(offset))
            .unwrap_or(false)
    }

    /// Set active bit for a serial number
    pub fn set_active_bit(&mut self, sn: u64, active: bool) {
        let twig_id = sn >> TWIG_SHIFT;
        let offset = (sn & ((1 << TWIG_SHIFT) - 1)) as u32;
        
        let bits = self.active_bits
            .entry(twig_id)
            .or_insert_with(ActiveBits::default);
        
        if active {
            bits.set_bit(offset);
        } else {
            bits.clear_bit(offset);
        }
        
        // Track which 512-bit positions were touched (for batch sync)
        self.touched_positions.insert(sn / 512);
    }
}
```

---

## Part 3: HybridIndexer Implementation

### 3.1 Replace BTreeMap with Unit-Based Indexer

The original uses a **HybridIndexer** with 65536 units for fine-grained locking:

```rust
//! HybridIndexer matching original QMDB design

use parking_lot::Mutex;
use std::sync::atomic::{AtomicUsize, Ordering};

/// Number of indexer units (2^16 = 65536)
pub const UNIT_COUNT: usize = 65536;
/// Units per shard
pub const UNITS_PER_SHARD: usize = UNIT_COUNT / 16; // 4096

/// Splits a 80-bit key into unit index and local key
#[inline]
pub fn split_k80(k80: &[u8]) -> (usize, u64) {
    let idx = u16::from_be_bytes([k80[0], k80[1]]) as usize;
    let k = u64::from_be_bytes(k80[2..10].try_into().unwrap());
    (idx, k)
}

/// A single indexer unit with its own mutex
pub struct Unit {
    /// Sorted list of (key, file_position) pairs
    entries: Vec<(u64, i64)>,
    /// Overlay of recent changes (unsorted)
    overlay: Vec<(u64, i64)>,
}

impl Unit {
    pub fn new() -> Self {
        Self {
            entries: Vec::new(),
            overlay: Vec::new(),
        }
    }

    pub fn add_kv(&mut self, k: u64, pos: i64) {
        self.overlay.push((k, pos));
    }

    pub fn find(&self, k: u64) -> Option<i64> {
        // Check overlay first (most recent)
        for &(key, pos) in self.overlay.iter().rev() {
            if key == k {
                return Some(pos);
            }
        }
        // Binary search in sorted entries
        self.entries
            .binary_search_by_key(&k, |&(key, _)| key)
            .ok()
            .map(|idx| self.entries[idx].1)
    }

    /// Merge overlay into sorted entries
    pub fn compact(&mut self) {
        if self.overlay.is_empty() {
            return;
        }
        self.overlay.sort_by_key(|&(k, _)| k);
        // Merge sorted overlay with entries
        let mut merged = Vec::with_capacity(self.entries.len() + self.overlay.len());
        let mut i = 0;
        let mut j = 0;
        while i < self.entries.len() && j < self.overlay.len() {
            if self.entries[i].0 <= self.overlay[j].0 {
                merged.push(self.entries[i]);
                i += 1;
            } else {
                merged.push(self.overlay[j]);
                j += 1;
            }
        }
        merged.extend_from_slice(&self.entries[i..]);
        merged.extend_from_slice(&self.overlay[j..]);
        self.entries = merged;
        self.overlay.clear();
    }
}

/// HybridIndexer with per-unit locking
pub struct HybridIndexer {
    units: Vec<Mutex<Unit>>,
    sizes: [AtomicUsize; 16], // per-shard size tracking
}

impl HybridIndexer {
    pub fn new() -> Self {
        let units = (0..UNIT_COUNT)
            .map(|_| Mutex::new(Unit::new()))
            .collect();
        Self {
            units,
            sizes: std::array::from_fn(|_| AtomicUsize::new(0)),
        }
    }

    pub fn add_kv(&self, k80: &[u8], pos: i64, _sn: u64) {
        let (idx, k) = split_k80(k80);
        let mut unit = self.units[idx].lock();
        unit.add_kv(k, pos);
        
        let shard_id = idx / UNITS_PER_SHARD;
        self.sizes[shard_id].fetch_add(1, Ordering::Relaxed);
    }

    pub fn find(&self, k80: &[u8]) -> Option<i64> {
        let (idx, k) = split_k80(k80);
        let unit = self.units[idx].lock();
        unit.find(k)
    }

    pub fn len(&self, shard_id: usize) -> usize {
        self.sizes[shard_id].load(Ordering::Relaxed)
    }
}
```

---

## Part 4: Pipeline Architecture

### 4.1 Component Overview

```
External Request
    |
    v
add_task(task_id) --> Prefetcher (1 main + N worker threads)
                            |
                            | [mid_sender per shard]
                            v
                      Updater[shard] (16 threads, one per shard)
                            |
                            | [EntryBuffer]
                            v
                      Flusher (1 main + 16 shard workers)
                            |
                            | [end_block_chan]
                            v
                      MetaInfo returned
```

### 4.2 Compactor Integration

Each shard has a dedicated compactor thread:

```rust
//! Background compaction matching original QMDB

use std::sync::Arc;
use std::thread;
use std::time::Duration;

pub struct Compactor {
    shard_id: usize,
    compact_trigger: usize,
    entry_file: Arc<EntryFile>,
    indexer: Arc<HybridIndexer>,
}

impl Compactor {
    pub fn start(self) {
        thread::spawn(move || {
            self.run();
        });
    }

    fn run(&self) {
        let mut file_pos = 0i64;
        loop {
            // Wait until we have enough entries to compact
            while self.indexer.len(self.shard_id) < self.compact_trigger {
                thread::sleep(Duration::from_millis(500));
            }

            // Read oldest active entry and re-append it
            // This moves it to the end, allowing old space to be reclaimed
            if let Some(entry) = self.read_oldest_active_entry(file_pos) {
                self.rewrite_entry(&entry);
                file_pos += entry.len() as i64;
            }
        }
    }
}
```

---

## Part 5: Memory Layout Optimization

### 5.1 Twig Eviction

Twigs are evicted to disk when all their active bits become zero:

```rust
impl<D: Digest> Shard<D> {
    /// Check and evict twigs with all-zero active bits
    pub fn evict_empty_twigs(&mut self) {
        let empty_twigs: Vec<u64> = self.active_bits
            .iter()
            .filter(|(_, bits)| bits.is_empty())
            .map(|(&id, _)| id)
            .collect();

        for twig_id in empty_twigs {
            // Move twig root to upper_tree nodes
            if let Some(twig) = self.twigs.remove(&twig_id) {
                self.upper_tree.set_twig_root(twig_id, twig.twig_root);
            }
            // Remove active bits (all zero anyway)
            self.active_bits.remove(&twig_id);
        }
    }
}
```

### 5.2 Memory Budget

| Component | Original QMDB | Target |
|-----------|--------------|--------|
| Active bits per twig | 256 bytes | 256 bytes |
| Twig metadata | ~320 bytes (5 hashes) | ~320 bytes |
| Youngest twig MT | 128 KB (4096 * 32) | 128 KB |
| Upper tree nodes | O(log N) * 32 bytes | O(log N) * 32 bytes |
| Indexer per unit | Variable | Same structure |

---

## Part 6: Implementation Phases

### Phase 1: Core Data Structures (2-3 weeks)

1. **Create twig module** (`storage/src/qmdb/twig.rs`)
   - `ActiveBits` struct with 256 bytes
   - `Twig` struct with 3-level right tree
   - Sync methods for each level

2. **Create upper tree** (`storage/src/qmdb/upper_tree.rs`)
   - Node position encoding
   - Edge node management for pruning
   - Level calculation

3. **Create shard** (`storage/src/qmdb/shard.rs`)
   - Per-shard state management
   - Active bit get/set with position tracking
   - Youngest twig merkle tree

### Phase 2: Indexer Replacement (2 weeks)

1. **Implement HybridIndexer** (`storage/src/index/hybrid.rs`)
   - 65536 unit structure
   - Per-unit mutex
   - Overlay + sorted entries

2. **Add background merger**
   - Periodic compaction of overlays
   - Size/change tracking per shard

### Phase 3: Pipeline Architecture (3-4 weeks)

1. **Implement Prefetcher**
   - io_uring integration (Linux)
   - Job management
   - Cache population

2. **Implement per-shard Updater**
   - Entry buffer writer
   - Index updates
   - Active bit management

3. **Implement Flusher**
   - Parallel per-shard flushing
   - Merkle tree sync
   - MetaDB updates

4. **Implement Compactor**
   - Background thread per shard
   - Entry rewriting
   - Utilization ratio tracking

### Phase 4: Integration & Testing (2 weeks)

1. **Cross-validation with original QMDB**
   - Generate test vectors from cyclone
   - Verify root hashes match
   - Verify proof generation matches

2. **Performance benchmarking**
   - Throughput comparison
   - Memory usage comparison
   - Latency distribution

---

## Part 7: Key Differences Summary

### Root Hash Calculation

**Original QMDB:**
```
TwigRoot = H(level=11 || left_root || active_bits_mtl3)
ShardRoot = H(level=max || child_left || child_right) [recursive up]
GlobalRoot = H(shard_roots[0] || ... || shard_roots[15])
```

**Current commonware QMDB:**
```
ChunkDigest = H(chunk_bytes)
MMR_Root = bag_peaks(chunk_digests)
PartialRoot = H(mmr_root || next_bit || last_chunk_digest)
GraftedRoot = H(operations_mmr || status_mmr)
```

### Constants Alignment

| Constant | Original | Commonware | Action |
|----------|----------|------------|--------|
| TWIG_SHIFT | 11 | N/A (uses 8) | Change to 11 |
| LEAF_COUNT_IN_TWIG | 2048 | 256 | Change to 2048 |
| SHARD_COUNT | 16 | 1 | Add 16 shards |
| UNIT_COUNT | 65536 | 1 | Add HybridIndexer |

---

## Part 8: Conformance Testing

### 8.1 Test Vector Generation

Generate test vectors from original QMDB:

```rust
#[test]
fn generate_compatibility_vectors() {
    // Use original QMDB to generate known-good hashes
    let mut qmdb = cyclone_qmdb::AdsWrap::new(&config);
    
    // Insert test entries
    for i in 0..10000 {
        qmdb.insert(key(i), value(i));
    }
    
    // Record root hashes at various points
    let roots = qmdb.get_all_shard_roots();
    
    // Generate proofs
    let proofs = qmdb.get_proofs(&test_keys);
    
    // Save to conformance.toml
}
```

### 8.2 Hash Compatibility

Add hash conformance tests:

```rust
#[test]
fn test_twig_root_compatibility() {
    // Known test case from original QMDB
    let active_bits = ActiveBits([0x12, 0x34, /* ... */ ]);
    let left_root = [0xAB; 32];
    
    let mut twig = Twig::default();
    twig.left_root = left_root;
    twig.sync_active_bits(&mut hasher, &active_bits, &[0, 1, 2, 3]);
    twig.sync_top(&mut hasher);
    
    // Compare with expected from original QMDB
    assert_eq!(
        hex::encode(twig.twig_root),
        "expected_hex_from_cyclone"
    );
}
```

---

## Part 9: Success Metrics

| Metric | Current | Target | Measurement |
|--------|---------|--------|-------------|
| Throughput | ~10% of original | 100% | ops/sec benchmark |
| Memory | ~5x original | 100% | RSS measurement |
| Proof generation | Baseline | 100% | Time to generate |
| Root hash | Incompatible | 100% compatible | Hash comparison |

---

## Appendix A: File Structure

```
storage/src/qmdb/
├── mod.rs              # Main module exports
├── twig.rs             # Twig + ActiveBits (NEW)
├── upper_tree.rs       # Upper tree management (NEW)
├── shard.rs            # Per-shard state (NEW)
├── pipeline/
│   ├── mod.rs
│   ├── prefetcher.rs   # io_uring prefetching (NEW)
│   ├── updater.rs      # Per-shard updater (NEW)
│   ├── flusher.rs      # Parallel flusher (NEW)
│   └── compactor.rs    # Background compaction (NEW)
├── current/
│   ├── mod.rs          # MODIFIED for twig-based root
│   └── ordered/
│       └── fixed.rs    # MODIFIED for new commit logic
└── any/
    └── mod.rs          # Floor management

storage/src/index/
├── mod.rs
├── ordered.rs          # Keep for compatibility
└── hybrid.rs           # HybridIndexer (NEW)
```

## Appendix B: Reference Implementation Links

Key files from [cyclone](https://github.com/LayerZero-Research/cyclone) to reference:

- `crates/qmdb/src/merkletree/tree.rs` - Tree and twig implementation
- `crates/qmdb/src/indexer/hybrid/mod.rs` - HybridIndexer
- `crates/qmdb-common/src/merkletree/twig.rs` - Twig structure
- `crates/qmdb-common/src/merkletree/activebits.rs` - ActiveBits
- `crates/qmdb-common/src/def.rs` - Constants and helpers
- `crates/qmdb/src/compactor.rs` - Background compaction
- `crates/qmdb/src/flusher/flusher.rs` - Parallel flushing
