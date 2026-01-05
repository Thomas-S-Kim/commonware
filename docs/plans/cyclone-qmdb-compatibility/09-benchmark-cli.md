# Task 9: Benchmark CLI Tool

## Current State

- No unified benchmark runner
- No standardized output format

## Goal

CLI tool (`qmdb-bench`) that runs workloads with configurable runtime and outputs metrics in multiple formats.

## Execution

### Step 1: CLI Structure

**File**: `tools/qmdb-bench/src/main.rs`

```rust
use clap::{Parser, ValueEnum};
use std::path::PathBuf;

#[derive(Parser, Debug)]
#[command(name = "qmdb-bench")]
struct Cli {
    #[arg(short, long)]
    workload: PathBuf,

    #[arg(short, long, default_value = "standard")]
    runtime: RuntimeBackend,

    #[arg(short, long, default_value = "mmr")]
    scheme: Scheme,

    #[arg(short, long, default_value = "text")]
    output: OutputFormat,

    #[arg(long)]
    push_gateway: Option<String>,

    #[arg(long, default_value = "qmdb_bench")]
    job_name: String,

    #[arg(long, default_value = "1")]
    warmup: u32,

    #[arg(long, default_value = "3")]
    iterations: u32,

    #[arg(short, long)]
    verbose: bool,
}

#[derive(Debug, Clone, ValueEnum)]
enum RuntimeBackend {
    Standard,
    IoUring,
}

#[derive(Debug, Clone, ValueEnum)]
enum Scheme {
    Mmr,
    BalancedTree,
}

#[derive(Debug, Clone, ValueEnum)]
enum OutputFormat {
    Text,
    Json,
    Prometheus,
}

fn main() -> anyhow::Result<()> {
    let cli = Cli::parse();
    let config = load_workload_config(&cli.workload)?;

    for _ in 0..cli.warmup {
        run_workload(&config, &cli)?;
    }

    let mut results = Vec::new();
    for _ in 0..cli.iterations {
        results.push(run_workload(&config, &cli)?);
    }

    let aggregated = aggregate_results(&results);
    match cli.output {
        OutputFormat::Text => output_text(&aggregated),
        OutputFormat::Json => output_json(&aggregated)?,
        OutputFormat::Prometheus => output_prometheus(&aggregated, &cli)?,
    }
    Ok(())
}
```

### Step 2: Output Formats

**File**: `tools/qmdb-bench/src/output.rs`

```rust
use prometheus::{Encoder, TextEncoder, GaugeVec, Opts, Registry};

pub fn output_text(result: &AggregatedResult) {
    println!("Workload: {}", result.name);
    println!("Ops/sec:  {:.2} (stddev: {:.2})", result.ops_per_sec.mean, result.ops_per_sec.stddev);
    println!("p50: {:?}, p95: {:?}, p99: {:?}", result.latencies.p50, result.latencies.p95, result.latencies.p99);
}

pub fn output_json(result: &AggregatedResult) -> anyhow::Result<()> {
    println!("{}", serde_json::to_string_pretty(result)?);
    Ok(())
}

pub fn output_prometheus(result: &AggregatedResult, cli: &Cli) -> anyhow::Result<()> {
    let registry = Registry::new();
    let ops_gauge = GaugeVec::new(
        Opts::new("qmdb_bench_ops_per_sec", "Operations per second"),
        &["workload", "scheme"],
    )?;
    registry.register(Box::new(ops_gauge.clone()))?;

    let scheme = format!("{:?}", cli.scheme);
    ops_gauge.with_label_values(&[&result.name, &scheme]).set(result.ops_per_sec.mean);

    if let Some(ref gateway) = cli.push_gateway {
        prometheus::push_metrics(&cli.job_name, prometheus::labels! {}, gateway, registry.gather(), None)?;
    } else {
        let encoder = TextEncoder::new();
        let mut buffer = Vec::new();
        encoder.encode(&registry.gather(), &mut buffer)?;
        println!("{}", String::from_utf8(buffer)?);
    }
    Ok(())
}
```

### Step 3: Cargo.toml

**File**: `tools/qmdb-bench/Cargo.toml`

```toml
[package]
name = "qmdb-bench"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "qmdb-bench"
path = "src/main.rs"

[dependencies]
commonware-storage = { path = "../../storage", features = ["balanced_tree"] }
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
```

## Files to Create

- `tools/qmdb-bench/Cargo.toml`
- `tools/qmdb-bench/src/main.rs`
- `tools/qmdb-bench/src/output.rs`
- `tools/qmdb-bench/src/workloads.rs`

## Files to Modify

- `Cargo.toml` (add workspace member)

## Usage

```bash
qmdb-bench --workload configs/write_sequential_1m.toml
qmdb-bench --workload configs/mixed_80_20.toml --runtime iouring --scheme balanced_tree
qmdb-bench --workload configs/mixed_80_20.toml --output prometheus --push-gateway http://localhost:9091
```

## Testing

```bash
cargo test -p qmdb-bench
cargo run -p qmdb-bench -- --workload configs/mixed_80_20.toml --iterations 1
```

## Dependencies

- Task 8 (workload definitions)
