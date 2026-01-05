# Task 8: Define Benchmark Workloads

## Current State

- No standardized workload definitions
- Ad-hoc benchmarks scattered across test files

## Goal

Reproducible workload configurations for QMDB benchmarking.

## Execution

### Step 1: Workload Types

| Workload | Description |
|----------|-------------|
| write_sequential | Sequential key inserts |
| write_random | Random key inserts |
| read_point | Point lookups |
| read_range | Range scans |
| mixed_80_20 | 80% reads, 20% writes |
| proof_gen | Inclusion proof generation |
| deactivation | Entry deactivation heavy |

### Step 2: Workload Trait

**File**: `tools/qmdb-bench/src/workloads/mod.rs`

```rust
#[derive(Debug, Clone, serde::Deserialize)]
pub struct WorkloadConfig {
    pub name: String,
    pub params: WorkloadParams,
    pub qmdb: QmdbConfig,
    pub runtime: RuntimeConfig,
}

#[derive(Debug, Clone, serde::Deserialize)]
#[serde(tag = "type")]
pub enum WorkloadParams {
    WriteSequential(WriteParams),
    WriteRandom(WriteParams),
    ReadPoint(ReadParams),
    Mixed(MixedParams),
    ProofGen(ProofParams),
}

#[derive(Debug, Clone, serde::Deserialize)]
pub struct WriteParams {
    pub operations: u64,
    pub key_size: usize,
    pub value_size: usize,
    pub batch_size: usize,
}

#[derive(Debug, Clone, serde::Deserialize)]
pub struct MixedParams {
    pub operations: u64,
    pub read_percent: u8,
    pub key_size: usize,
    pub value_size: usize,
}

#[derive(Debug, Clone, serde::Deserialize)]
pub enum Scheme {
    MmrScheme,
    BalancedTreeScheme,
}

pub trait Workload: Send {
    fn run(&self, config: &WorkloadConfig) -> WorkloadResult;
    fn name(&self) -> &str;
}

#[derive(Debug, Clone)]
pub struct WorkloadResult {
    pub name: String,
    pub duration: Duration,
    pub ops_per_sec: f64,
    pub latencies: LatencyStats,
}

#[derive(Debug, Clone, Default)]
pub struct LatencyStats {
    pub p50: Duration,
    pub p95: Duration,
    pub p99: Duration,
    pub max: Duration,
}
```

### Step 3: Configuration Files

**File**: `tools/qmdb-bench/configs/write_sequential_1m.toml`

```toml
name = "write_sequential_1m"

[params]
type = "WriteSequential"
operations = 1_000_000
key_size = 32
value_size = 256
batch_size = 1000

[qmdb]
mode = "Fresh"
scheme = "MmrScheme"

[runtime]
backend = "IoUring"
thread_pool_size = 4
```

**File**: `tools/qmdb-bench/configs/mixed_80_20.toml`

```toml
name = "mixed_80_20"

[params]
type = "Mixed"
operations = 1_000_000
read_percent = 80
key_size = 32
value_size = 256

[qmdb]
mode = "Fresh"
scheme = "MmrScheme"

[runtime]
backend = "IoUring"
thread_pool_size = 4
```

## Files to Create

- `tools/qmdb-bench/src/workloads/mod.rs`
- `tools/qmdb-bench/src/workloads/write_heavy.rs`
- `tools/qmdb-bench/src/workloads/mixed.rs`
- `tools/qmdb-bench/configs/*.toml`

## Testing

```bash
cargo test -p qmdb-bench workload
```

## Dependencies

- None (parallel with compatibility work)
