# Comparison: High-Performance QMDB vs. Simplified MMR-QMDB

This document summarizes the architectural differences and performance bottlenecks discovered when comparing the original high-performance QMDB (`crates/qmdb`) with the simplified MMR-based version (`3rd_party/monorepo-main/storage/src/qmdb`).

**Scope note (important):** the analysis below focuses on the simplified implementation’s **`qmdb/current/ordered`** path (used for ordered/exclusion proofs) with an **order-preserving capped-key translator (e.g., `EightCap`)**. The evaluation environment is **Linux**, and the runtime is configured such that **io_uring is available** at the storage layer.

The evaluation shows that the simplified version achieves only **10% of the throughput** while consuming **5 times more memory** than the high-performance version.

---

## 1. Concurrency Model: Pipeline vs. Single-DB Serialization

| Feature | High-Performance QMDB | Simplified MMR-QMDB (`current/ordered`) |
| :--- | :--- | :--- |
| **Concurrency** | **Shard-parallel + pipeline overlap** (fine-grained synchronization) | **Serialized mutation path** (mutable `&mut self` APIs; typically wrapped by `RwLock` for sharing) |
| **Architecture** | 16-shard streaming pipeline | Single logical DB per instance (async, but not pipelined into independent components) |
| **Mechanism** | `EntryBuffer` decouples writers from flushing / Merkle updates | **Commit-path Heavy**: log sync, bitmap merkleization, and pruning happen synchronously |
| **Parallelism** | 50+ dedicated threads (Prefetcher, Updater, Flusher, etc.) | Standard async tasks with **Local parallelism only** (optional ThreadPool for bitmap) |

**Insight:** The high-performance version decouples write operations from disk flushing and Merkle tree updates using a 2-block latency pipeline. The simplified version, while supporting some parallel hashing for its bitmap, still serializes the core logic. The critical path is blocked by synchronous commit operations, preventing the overlap of I/O and computation across blocks.

---

## 2. Authenticated Structure: Twig-based vs. (Log MMR + Authenticated Status BitMap)

| Feature | High-Performance QMDB | Simplified MMR-QMDB (`current/ordered`) |
| :--- | :--- | :--- |
| **Storage Strategy** | **Evict-to-Disk**: old Twig structures are persisted; only compact summaries + recent working set stay hot | **Window-driven memory**: memory grows with the range between floor and tip |
| **ADS Structure** | Twig-based Merkle (Twig roots + Top tree) | Authenticated log (MMR-backed journal) + AuthenticatedBitMap (MMR-over-bitmap) |
| **Active Status** | Compact `ActiveBits` using raw atomic bits (`AtomicU32`) | `AuthenticatedBitMap`: maintains **full Merkle node digests** in memory for the active range |
| **Memory Scaling** | Bounded by shard working set (no redundant internal hashes in RAM) | High overhead due to **in-memory MMR node redundancy** (`VecDeque<D>`) |

**Insight:** High-performance QMDB avoids keeping Merkle nodes in memory, storing only raw bits (`ActiveBits`) and calculating hashes on-the-fly during flush. The simplified version's `AuthenticatedBitMap` maintains a full in-memory MMR tree structure for the active portion of the log. Even with pruning (`prune_to_bit`), the constant factor of storing 32-byte digests for every few bits leads to the observed 5x memory inflation.

---

## 3. Compaction & Commit Strategy: Smooth Pipeline vs. Synchronous Jitter

| Feature | High-Performance QMDB | Simplified MMR-QMDB |
| :--- | :--- | :--- |
| **Execution** | Background `Compactor` threads per shard | Synchronous `raise_floor` during `commit` |
| **Trigger** | Amortized cost integrated into C/U/D pipeline | Batch processing of "steps" at block end |
| **Commit Payload** | Lightweight (signals metadata + root) | **Heavyweight**: `steps+1` floor moves + Merkle Sync + Prune |
| **Impact** | Transparent to the user, zero jitter | **Significant Latency Spikes** (Tail Latency) during commit |

**Insight:** Simplified QMDB concentrates all maintenance work (advancing the inactivity floor, re-merkleizing the bitmap, and pruning nodes) into the `commit` phase. This "concentration of work" creates CPU and I/O bursts. In contrast, original QMDB spreads this load across the background pipeline, ensuring smooth throughput and predictable response times.

---

## 4. Indexer & Locking Depth

| Feature | High-Performance QMDB | Simplified MMR-QMDB (`current/ordered`) |
| :--- | :--- | :--- |
| **Implementation** |  `HybridIndexer` and LSM-style `InMemIndexer`  | **Single BTreeMap Index** |
| **Sharding** | **Two-level**: Shard-level + Unit-level (fine-grained) | **None** (Global index instance) |
| **Locking** | Per-Unit `Mutex` + Lock-free bit segments | Single `&mut self` or Global `RwLock` |
| **Burden** | Hash-based point lookups | Ordered metadata (next-key) for exclusion proofs |

**Insight:** The high-performance version uses二级分片 (Two-level sharding), mapping keys to specific `Units` with independent locks, which reduces contention to near zero. The simplified version uses a single `BTreeMap` for all keys; this global structure becomes a massive bottleneck under high concurrency, especially since maintaining the "ordered" property for exclusion proofs is more computationally expensive than the original QMDB's sharded hash approach.

---

## 5. I/O & Caching Model

| Feature | High-Performance QMDB | Simplified MMR-QMDB (`current/ordered`) |
| :--- | :--- | :--- |
| **I/O Backend** | Explicit io_uring path in entry I/O + prefetching | Generic runtime storage abstraction |
| **Caching** | Workload-specific `EntryCache` per shard | Generic buffer pools / page caches |
| **Prefetching** | Dedicated `Prefetcher` pipeline | No specialized prefetching logic |

**Insight:** Original QMDB's `Prefetcher` is designed for the blockchain "Read-Modify-Write" pattern, warming up the `EntryCache` before the `Updater` starts. The simplified version relies on general-purpose runtime caches, which suffer from higher read amplification and cannot hide disk latency as effectively as a dedicated prefetch pipeline.

---

## 6. Conclusion: Engineering Efficiency vs. Minimal Implementation

The simplified MMR-QMDB prioritizes "obvious correctness" and "minimal implementation" over engineering efficiency. While it provides the same functional guarantees, its performance is crippled by:

1.  **Lack of Sharding & Pipelining**: No overlap of block execution and block finalization (Merkle/IO).
2.  **Redundant Memory State**: Storing intermediate Merkle digests in RAM instead of just raw bits.
3.  **Synchronous Commit Burdens**: Forcing maintenance tasks (`raise_floor`, `merkleize`, `prune`) onto the critical path.
4.  **Global Lock Contention**: Using a single `BTreeMap` instead of a sharded, unit-level architecture.

These architectural choices compound multiplicatively, resulting in the measured 10x throughput reduction and 5x memory overhead.
