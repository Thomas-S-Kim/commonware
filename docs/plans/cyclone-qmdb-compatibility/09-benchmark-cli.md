# Task 9: Build Benchmark CLI Tool

## Current State

- No unified benchmark runner for QMDB
- Cannot easily run workloads with different runtime configurations
- No standardized output format for metrics

## Expected Goal

A CLI tool (`qmdb-bench`) that runs workloads against QMDB with configurable runtime options and outputs metrics in multiple formats.

## Rationale

A dedicated CLI tool:
1. **Standardization**: Consistent interface for all benchmarks
2. **Flexibility**: Easy to switch runtime backends and configs
3. **Integration**: Can push metrics to Prometheus for tracking
4. **Comparison**: Compare native vs cyclone-compat modes

## Execution Plan

### Step 1: Create CLI Structure

**File**: `tools/qmdb-bench/src/main.rs`

```rust
//! QMDB Benchmark CLI
//!
//! Run workloads against QMDB with configurable runtime and output options.

use clap::{Parser, ValueEnum};
use std::path::PathBuf;

#[derive(Parser, Debug)]
#[command(name = "qmdb-bench")]
#[command(about = "Benchmark tool for commonware-storage QMDB")]
struct Cli {
    /// Path to workload configuration file
    #[arg(short, long)]
    workload: PathBuf,

    /// I/O backend to use
    #[arg(short, long, default_value = "standard")]
    runtime: RuntimeBackend,

    /// Compatibility mode
    #[arg(short, long, default_value = "native")]
    mode: CompatMode,

    /// Output format
    #[arg(short, long, default_value = "text")]
    output: OutputFormat,

    /// Prometheus push gateway URL (for prometheus output)
    #[arg(long)]
    push_gateway: Option<String>,

    /// Job name for Prometheus metrics
    #[arg(long, default_value = "qmdb_bench")]
    job_name: String,

    /// Number of warmup iterations
    #[arg(long, default_value = "1")]
    warmup: u32,

    /// Number of measurement iterations
    #[arg(long, default_value = "3")]
    iterations: u32,

    /// Verbose output
    #[arg(short, long)]
    verbose: bool,
}

#[derive(Debug, Clone, ValueEnum)]
enum RuntimeBackend {
    Standard,
    IoUring,
}

#[derive(Debug, Clone, ValueEnum)]
enum CompatMode {
    Native,
    Cyclone,
}

#[derive(Debug, Clone, ValueEnum)]
enum OutputFormat {
    Text,
    Json,
    Prometheus,
}

fn main() -> anyhow::Result<()> {
    let cli = Cli::parse();

    // Initialize logging
    if cli.verbose {
        tracing_subscriber::fmt::init();
    }

    // Load workload config
    let config = load_workload_config(&cli.workload)?;

    // Run warmup iterations
    if cli.warmup > 0 {
        println!("Running {} warmup iteration(s)...", cli.warmup);
        for _ in 0..cli.warmup {
            run_workload(&config, &cli)?;
        }
    }

    // Run measurement iterations
    println!("Running {} measurement iteration(s)...", cli.iterations);
    let mut results = Vec::new();
    for i in 0..cli.iterations {
        let result = run_workload(&config, &cli)?;
        results.push(result);
        if cli.verbose {
            println!("  Iteration {}: {:.2} ops/sec", i + 1, result.ops_per_sec);
        }
    }

    // Aggregate results
    let aggregated = aggregate_results(&results);

    // Output results
    match cli.output {
        OutputFormat::Text => output_text(&aggregated),
        OutputFormat::Json => output_json(&aggregated)?,
        OutputFormat::Prometheus => output_prometheus(&aggregated, &cli)?,
    }

    Ok(())
}

fn load_workload_config(path: &PathBuf) -> anyhow::Result<WorkloadConfig> {
    let content = std::fs::read_to_string(path)?;
    let config: WorkloadConfig = toml::from_str(&content)?;
    Ok(config)
}

fn run_workload(config: &WorkloadConfig, cli: &Cli) -> anyhow::Result<WorkloadResult> {
    // Override config with CLI options
    let mut config = config.clone();
    config.runtime.backend = match cli.runtime {
        RuntimeBackend::Standard => IoBackend::Standard,
        RuntimeBackend::IoUring => IoBackend::IoUring,
    };
    config.qmdb.compat_mode = match cli.mode {
        CompatMode::Native => qmdb_bench::CompatMode::Native,
        CompatMode::Cyclone => qmdb_bench::CompatMode::Cyclone,
    };

    // Create and run workload
    let workload = create_workload(&config.params)?;
    let result = workload.run(&config);

    Ok(result)
}
```

### Step 2: Implement Runtime Integration

**File**: `tools/qmdb-bench/src/runtime.rs`

```rust
//! Runtime setup and configuration.

use commonware_runtime::{Runner, deterministic, tokio as cw_tokio};

pub enum RuntimeHandle {
    Tokio(tokio::runtime::Runtime),
    Deterministic(deterministic::Runner),
}

pub fn create_runtime(config: &RuntimeConfig) -> anyhow::Result<RuntimeHandle> {
    match config.backend {
        IoBackend::Standard => {
            let rt = tokio::runtime::Builder::new_multi_thread()
                .worker_threads(config.thread_pool_size)
                .enable_all()
                .build()?;
            Ok(RuntimeHandle::Tokio(rt))
        }
        IoBackend::IoUring => {
            #[cfg(target_os = "linux")]
            {
                // Configure io_uring runtime
                let rt = tokio::runtime::Builder::new_multi_thread()
                    .worker_threads(config.thread_pool_size)
                    .enable_all()
                    .build()?;
                Ok(RuntimeHandle::Tokio(rt))
            }
            #[cfg(not(target_os = "linux"))]
            {
                anyhow::bail!("io_uring is only supported on Linux");
            }
        }
    }
}
```

### Step 3: Implement Output Formats

**File**: `tools/qmdb-bench/src/output.rs`

```rust
//! Output format implementations.

use crate::WorkloadResult;
use prometheus::{Encoder, TextEncoder, GaugeVec, Opts, Registry};

pub fn output_text(result: &AggregatedResult) {
    println!("\n=== Benchmark Results ===\n");
    println!("Workload: {}", result.name);
    println!("Iterations: {}", result.iterations);
    println!();
    println!("Throughput:");
    println!("  Mean:   {:.2} ops/sec", result.ops_per_sec.mean);
    println!("  StdDev: {:.2} ops/sec", result.ops_per_sec.stddev);
    println!();
    println!("Latency:");
    println!("  p50:  {:?}", result.latencies.p50);
    println!("  p95:  {:?}", result.latencies.p95);
    println!("  p99:  {:?}", result.latencies.p99);
    println!("  max:  {:?}", result.latencies.max);
}

pub fn output_json(result: &AggregatedResult) -> anyhow::Result<()> {
    let json = serde_json::to_string_pretty(result)?;
    println!("{}", json);
    Ok(())
}

pub fn output_prometheus(result: &AggregatedResult, cli: &Cli) -> anyhow::Result<()> {
    let registry = Registry::new();

    // Create metrics
    let ops_gauge = GaugeVec::new(
        Opts::new("qmdb_bench_ops_per_sec", "Operations per second"),
        &["workload", "mode"],
    )?;
    registry.register(Box::new(ops_gauge.clone()))?;

    let latency_gauge = GaugeVec::new(
        Opts::new("qmdb_bench_latency_seconds", "Latency in seconds"),
        &["workload", "mode", "percentile"],
    )?;
    registry.register(Box::new(latency_gauge.clone()))?;

    // Set values
    let mode = format!("{:?}", cli.mode);
    ops_gauge
        .with_label_values(&[&result.name, &mode])
        .set(result.ops_per_sec.mean);

    latency_gauge
        .with_label_values(&[&result.name, &mode, "p50"])
        .set(result.latencies.p50.as_secs_f64());
    latency_gauge
        .with_label_values(&[&result.name, &mode, "p95"])
        .set(result.latencies.p95.as_secs_f64());
    latency_gauge
        .with_label_values(&[&result.name, &mode, "p99"])
        .set(result.latencies.p99.as_secs_f64());

    // Push to gateway or print
    if let Some(ref gateway) = cli.push_gateway {
        push_to_gateway(gateway, &cli.job_name, &registry)?;
    } else {
        let encoder = TextEncoder::new();
        let mut buffer = Vec::new();
        encoder.encode(&registry.gather(), &mut buffer)?;
        println!("{}", String::from_utf8(buffer)?);
    }

    Ok(())
}

fn push_to_gateway(gateway: &str, job: &str, registry: &Registry) -> anyhow::Result<()> {
    prometheus::push_metrics(
        job,
        prometheus::labels! {},
        gateway,
        registry.gather(),
        None,
    )?;
    Ok(())
}
```

### Step 4: Create Cargo.toml

**File**: `tools/qmdb-bench/Cargo.toml`

```toml
[package]
name = "qmdb-bench"
version = "0.1.0"
edition = "2021"
description = "Benchmark tool for commonware-storage QMDB"

[[bin]]
name = "qmdb-bench"
path = "src/main.rs"

[dependencies]
commonware-storage = { path = "../../storage", features = ["cyclone-compat"] }
commonware-runtime = { path = "../../runtime" }

clap = { version = "4", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
anyhow = "1"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
toml = "0.8"
tracing = "0.1"
tracing-subscriber = "0.3"
prometheus = "0.13"
rand = "0.8"
rand_chacha = "0.3"

[target.'cfg(target_os = "linux")'.dependencies]
# io_uring dependencies if needed
```

## Files to Create

- `tools/qmdb-bench/Cargo.toml`
- `tools/qmdb-bench/src/main.rs`
- `tools/qmdb-bench/src/lib.rs`
- `tools/qmdb-bench/src/runtime.rs`
- `tools/qmdb-bench/src/output.rs`
- `tools/qmdb-bench/src/workloads.rs`

## Files to Modify

- `Cargo.toml` (add workspace member)

## Example Usage

```bash
# Run write workload with text output
qmdb-bench --workload configs/write_sequential_1m.toml

# Run with io_uring backend
qmdb-bench --workload configs/write_sequential_1m.toml --runtime iouring

# Compare native vs cyclone
qmdb-bench --workload configs/mixed_80_20.toml --mode native
qmdb-bench --workload configs/mixed_80_20.toml --mode cyclone

# Push metrics to Prometheus
qmdb-bench --workload configs/full_suite.toml \
           --output prometheus \
           --push-gateway http://localhost:9091

# Verbose output with multiple iterations
qmdb-bench --workload configs/mixed_80_20.toml \
           --verbose \
           --warmup 2 \
           --iterations 5
```

## Testing Strategy

- Unit tests for config parsing and output formatting
- Integration test that runs a small workload end-to-end
- Test Prometheus metric generation

## Dependencies

- Task 8 (workload definitions)

## Estimated Effort

- 2-3 days implementation
- 0.5 day testing
