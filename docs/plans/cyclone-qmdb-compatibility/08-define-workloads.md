# Task 8: Define QMDB Benchmark Workloads

## Current State

- No standardized workload definitions for QMDB benchmarking
- Ad-hoc benchmarks scattered across test files
- No reproducible benchmark configurations

## Expected Goal

A comprehensive suite of representative workloads for benchmarking QMDB performance, defined as reproducible configurations that can be run consistently across different environments.

## Rationale

Standardized workloads enable:
1. **Reproducibility**: Same workload produces comparable results
2. **Coverage**: Test different access patterns and scales
3. **Comparison**: Fair comparison between implementations
4. **Regression detection**: Detect performance changes over time

## Execution Plan

### Step 1: Define Workload Types

Create workload definitions covering different access patterns:

| Workload | Description | Key Metrics |
|----------|-------------|-------------|
| write_sequential | Sequential key inserts | Throughput, latency |
| write_random | Random key inserts | Throughput, latency |
| read_point | Point lookups by key | Latency p50/p95/p99 |
| read_range | Range scans | Throughput, latency |
| mixed_80_20 | 80% reads, 20% writes | Combined throughput |
| mixed_50_50 | 50% reads, 50% writes | Combined throughput |
| proof_gen | Inclusion proof generation | Proof time, size |
| compaction | Writes with compaction | Write amplification |
| deactivation | Entry deactivation heavy | Throughput after deactivate |

### Step 2: Create Workload Trait

**File**: `storage/benches/workloads/mod.rs`

```rust
//! QMDB benchmark workload definitions.

use std::time::Duration;

pub mod write_heavy;
pub mod read_heavy;
pub mod mixed;
pub mod proof_gen;

/// Configuration for a benchmark workload.
#[derive(Debug, Clone, serde::Deserialize)]
pub struct WorkloadConfig {
    /// Human-readable name.
    pub name: String,

    /// Description of what this workload tests.
    pub description: String,

    /// Workload-specific parameters.
    pub params: WorkloadParams,

    /// QMDB configuration.
    pub qmdb: QmdbConfig,

    /// Runtime configuration.
    pub runtime: RuntimeConfig,
}

/// Workload-specific parameters.
#[derive(Debug, Clone, serde::Deserialize)]
#[serde(tag = "type")]
pub enum WorkloadParams {
    WriteSequential(WriteParams),
    WriteRandom(WriteParams),
    ReadPoint(ReadParams),
    ReadRange(ReadParams),
    Mixed(MixedParams),
    ProofGen(ProofParams),
}

#[derive(Debug, Clone, serde::Deserialize)]
pub struct WriteParams {
    /// Total number of operations.
    pub operations: u64,
    /// Key size in bytes.
    pub key_size: usize,
    /// Value size in bytes.
    pub value_size: usize,
    /// Batch size for bulk operations.
    pub batch_size: usize,
}

#[derive(Debug, Clone, serde::Deserialize)]
pub struct ReadParams {
    /// Number of entries to pre-populate.
    pub initial_entries: u64,
    /// Number of read operations.
    pub operations: u64,
    /// For range reads: range size.
    pub range_size: Option<usize>,
}

#[derive(Debug, Clone, serde::Deserialize)]
pub struct MixedParams {
    /// Total operations.
    pub operations: u64,
    /// Read percentage (0-100).
    pub read_percent: u8,
    /// Key size in bytes.
    pub key_size: usize,
    /// Value size in bytes.
    pub value_size: usize,
}

#[derive(Debug, Clone, serde::Deserialize)]
pub struct ProofParams {
    /// Number of entries to populate.
    pub entries: u64,
    /// Number of proofs to generate.
    pub proofs: u64,
}

#[derive(Debug, Clone, serde::Deserialize)]
pub struct QmdbConfig {
    /// Start with fresh or existing database.
    pub mode: DbMode,
    /// Path for existing database.
    pub path: Option<String>,
    /// Compatibility mode.
    pub compat_mode: CompatMode,
}

#[derive(Debug, Clone, serde::Deserialize)]
pub enum DbMode {
    Fresh,
    Existing,
}

#[derive(Debug, Clone, serde::Deserialize)]
pub enum CompatMode {
    Native,
    Cyclone,
}

#[derive(Debug, Clone, serde::Deserialize)]
pub struct RuntimeConfig {
    /// I/O backend.
    pub backend: IoBackend,
    /// Thread pool size.
    pub thread_pool_size: usize,
}

#[derive(Debug, Clone, serde::Deserialize)]
pub enum IoBackend {
    Standard,
    IoUring,
}

/// Trait for executable workloads.
pub trait Workload: Send {
    /// Run the workload and return results.
    fn run(&self, config: &WorkloadConfig) -> WorkloadResult;

    /// Get workload name.
    fn name(&self) -> &str;
}

/// Results from running a workload.
#[derive(Debug, Clone)]
pub struct WorkloadResult {
    /// Workload name.
    pub name: String,
    /// Total duration.
    pub duration: Duration,
    /// Operations per second.
    pub ops_per_sec: f64,
    /// Bytes per second (if applicable).
    pub bytes_per_sec: Option<f64>,
    /// Latency percentiles.
    pub latencies: LatencyStats,
    /// Additional metrics.
    pub extra: std::collections::HashMap<String, f64>,
}

#[derive(Debug, Clone, Default)]
pub struct LatencyStats {
    pub p50: Duration,
    pub p95: Duration,
    pub p99: Duration,
    pub max: Duration,
}
```

### Step 3: Implement Write Workloads

**File**: `storage/benches/workloads/write_heavy.rs`

```rust
//! Write-heavy workload implementations.

use super::*;
use rand::{Rng, SeedableRng};
use rand_chacha::ChaCha8Rng;

pub struct WriteSequential;

impl Workload for WriteSequential {
    fn name(&self) -> &str {
        "write_sequential"
    }

    fn run(&self, config: &WorkloadConfig) -> WorkloadResult {
        let WorkloadParams::WriteSequential(params) = &config.params else {
            panic!("wrong params type");
        };

        // Implementation
        todo!()
    }
}

pub struct WriteRandom;

impl Workload for WriteRandom {
    fn name(&self) -> &str {
        "write_random"
    }

    fn run(&self, config: &WorkloadConfig) -> WorkloadResult {
        let WorkloadParams::WriteRandom(params) = &config.params else {
            panic!("wrong params type");
        };

        // Use deterministic RNG for reproducibility
        let mut rng = ChaCha8Rng::seed_from_u64(42);

        // Implementation
        todo!()
    }
}
```

### Step 4: Create Configuration Files

**File**: `storage/benches/configs/write_sequential_1m.toml`

```toml
name = "write_sequential_1m"
description = "1 million sequential writes with 32-byte keys and 256-byte values"

[params]
type = "WriteSequential"
operations = 1_000_000
key_size = 32
value_size = 256
batch_size = 1000

[qmdb]
mode = "Fresh"
compat_mode = "Native"

[runtime]
backend = "IoUring"
thread_pool_size = 4
```

**File**: `storage/benches/configs/mixed_80_20.toml`

```toml
name = "mixed_80_20"
description = "80% reads, 20% writes with 1M pre-populated entries"

[params]
type = "Mixed"
operations = 1_000_000
read_percent = 80
key_size = 32
value_size = 256

[qmdb]
mode = "Fresh"
compat_mode = "Native"

[runtime]
backend = "IoUring"
thread_pool_size = 4
```

**File**: `storage/benches/configs/proof_gen_10k.toml`

```toml
name = "proof_gen_10k"
description = "Generate 10K inclusion proofs from 1M entry database"

[params]
type = "ProofGen"
entries = 1_000_000
proofs = 10_000

[qmdb]
mode = "Fresh"
compat_mode = "Native"

[runtime]
backend = "IoUring"
thread_pool_size = 4
```

### Step 5: Add Cyclone Comparison Configs

**File**: `storage/benches/configs/compare_native_vs_cyclone.toml`

```toml
name = "compare_native_vs_cyclone"
description = "Compare native vs cyclone-compat mode for mixed workload"

# This config is used twice: once with Native, once with Cyclone
[params]
type = "Mixed"
operations = 100_000
read_percent = 50
key_size = 32
value_size = 256

[qmdb]
mode = "Fresh"
# compat_mode set by CLI

[runtime]
backend = "IoUring"
thread_pool_size = 4
```

## Files to Create

- `storage/benches/workloads/mod.rs`
- `storage/benches/workloads/write_heavy.rs`
- `storage/benches/workloads/read_heavy.rs`
- `storage/benches/workloads/mixed.rs`
- `storage/benches/workloads/proof_gen.rs`
- `storage/benches/configs/*.toml` (multiple config files)

## Files to Modify

- `storage/Cargo.toml` (add bench dependencies: serde, toml, rand, rand_chacha)

## Testing Strategy

- Unit tests for config parsing
- Smoke tests that run each workload for 1 second
- Validate deterministic RNG produces reproducible results

## Dependencies

- None (can be done in parallel with compatibility work)

## Estimated Effort

- 1-2 days for workload trait and implementations
- 0.5 day for config files
- 0.5 day for testing
