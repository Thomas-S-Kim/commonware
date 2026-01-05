# Task 10: Audit and Extend Storage Telemetry

## Current State

- Some telemetry exists in commonware-storage but coverage is incomplete
- Gaps in visibility for key performance indicators
- No comprehensive documentation of available metrics

## Expected Goal

Comprehensive metrics coverage for performance analysis, with all metrics Prometheus-compatible and well-documented.

## Rationale

Complete telemetry enables:
1. **Visibility**: Understand where time is spent
2. **Debugging**: Identify bottlenecks and anomalies
3. **Comparison**: Compare native vs cyclone-compat performance
4. **Alerting**: Detect performance regressions

## Execution Plan

### Step 1: Audit Existing Metrics

Review current telemetry in:
- `storage/src/mmr/*.rs`
- `storage/src/bitmap/*.rs`
- `storage/src/qmdb/*.rs`
- `storage/src/journal/*.rs`

Document findings:

| Component | Existing Metrics | Missing Metrics |
|-----------|------------------|-----------------|
| MMR | ? | ? |
| Bitmap | ? | ? |
| QMDB | ? | ? |
| Journal | ? | ? |

### Step 2: Define Required Metrics

**Operation Latencies:**
```
qmdb_operation_duration_seconds{operation="add", quantile="0.5|0.95|0.99"}
qmdb_operation_duration_seconds{operation="get", quantile="..."}
qmdb_operation_duration_seconds{operation="proof", quantile="..."}
qmdb_operation_duration_seconds{operation="commit", quantile="..."}
```

**Throughput:**
```
qmdb_operations_total{operation="add|get|proof|commit"}
qmdb_bytes_total{direction="read|write"}
```

**Cache Metrics:**
```
qmdb_cache_hits_total
qmdb_cache_misses_total
qmdb_cache_size_bytes
```

**Tree Metrics:**
```
qmdb_mmr_size
qmdb_mmr_peaks
qmdb_bitmap_size_bits
qmdb_twig_count (cyclone-compat)
```

**I/O Metrics:**
```
qmdb_io_operations_total{operation="read|write|sync"}
qmdb_io_bytes_total{direction="read|write"}
qmdb_io_wait_seconds_total
```

**Compaction Metrics:**
```
qmdb_compaction_duration_seconds
qmdb_compaction_entries_processed
qmdb_compaction_bytes_reclaimed
```

### Step 3: Implement Metric Collection

**File**: `storage/src/metrics.rs`

```rust
//! Prometheus metrics for commonware-storage.

use prometheus::{
    register_counter_vec, register_gauge, register_gauge_vec,
    register_histogram_vec, CounterVec, Gauge, GaugeVec, HistogramVec,
};
use once_cell::sync::Lazy;

// Operation latencies
pub static OPERATION_DURATION: Lazy<HistogramVec> = Lazy::new(|| {
    register_histogram_vec!(
        "qmdb_operation_duration_seconds",
        "Duration of QMDB operations",
        &["operation"],
        vec![0.0001, 0.0005, 0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0]
    )
    .unwrap()
});

// Operation counts
pub static OPERATIONS_TOTAL: Lazy<CounterVec> = Lazy::new(|| {
    register_counter_vec!(
        "qmdb_operations_total",
        "Total QMDB operations",
        &["operation"]
    )
    .unwrap()
});

// Bytes transferred
pub static BYTES_TOTAL: Lazy<CounterVec> = Lazy::new(|| {
    register_counter_vec!(
        "qmdb_bytes_total",
        "Total bytes read/written",
        &["direction"]
    )
    .unwrap()
});

// Cache metrics
pub static CACHE_HITS: Lazy<Counter> = Lazy::new(|| {
    register_counter!("qmdb_cache_hits_total", "Cache hits").unwrap()
});

pub static CACHE_MISSES: Lazy<Counter> = Lazy::new(|| {
    register_counter!("qmdb_cache_misses_total", "Cache misses").unwrap()
});

pub static CACHE_SIZE: Lazy<Gauge> = Lazy::new(|| {
    register_gauge!("qmdb_cache_size_bytes", "Cache size in bytes").unwrap()
});

// Tree metrics
pub static MMR_SIZE: Lazy<Gauge> = Lazy::new(|| {
    register_gauge!("qmdb_mmr_size", "MMR size (node count)").unwrap()
});

pub static BITMAP_SIZE: Lazy<Gauge> = Lazy::new(|| {
    register_gauge!("qmdb_bitmap_size_bits", "Bitmap size in bits").unwrap()
});

// Cyclone-compat metrics
#[cfg(feature = "cyclone-compat")]
pub static TWIG_COUNT: Lazy<Gauge> = Lazy::new(|| {
    register_gauge!("qmdb_twig_count", "Number of twigs (cyclone-compat)").unwrap()
});

/// Helper to time an operation and record metrics.
pub struct OperationTimer {
    operation: &'static str,
    start: std::time::Instant,
}

impl OperationTimer {
    pub fn new(operation: &'static str) -> Self {
        Self {
            operation,
            start: std::time::Instant::now(),
        }
    }
}

impl Drop for OperationTimer {
    fn drop(&mut self) {
        let duration = self.start.elapsed();
        OPERATION_DURATION
            .with_label_values(&[self.operation])
            .observe(duration.as_secs_f64());
        OPERATIONS_TOTAL
            .with_label_values(&[self.operation])
            .inc();
    }
}

/// Macro for timing operations.
#[macro_export]
macro_rules! time_operation {
    ($op:expr, $body:expr) => {{
        let _timer = $crate::metrics::OperationTimer::new($op);
        $body
    }};
}
```

### Step 4: Instrument Key Code Paths

**File**: `storage/src/qmdb/current/mod.rs` (example)

```rust
use crate::metrics::{time_operation, OPERATIONS_TOTAL, BYTES_TOTAL};

impl<...> CurrentDb<...> {
    pub async fn add(&mut self, key: &[u8], value: &[u8]) -> Result<u64, Error> {
        time_operation!("add", {
            BYTES_TOTAL
                .with_label_values(&["write"])
                .inc_by((key.len() + value.len()) as f64);

            // ... existing implementation ...
        })
    }

    pub async fn get(&self, key: &[u8]) -> Result<Option<Vec<u8>>, Error> {
        time_operation!("get", {
            // Check cache first
            if let Some(value) = self.cache.get(key) {
                CACHE_HITS.inc();
                return Ok(Some(value));
            }
            CACHE_MISSES.inc();

            // ... existing implementation ...
        })
    }
}
```

### Step 5: Document Metrics

**File**: `storage/src/metrics.rs` (add documentation section)

```rust
//! # Available Metrics
//!
//! ## Operation Metrics
//!
//! | Metric | Type | Labels | Description |
//! |--------|------|--------|-------------|
//! | `qmdb_operation_duration_seconds` | Histogram | operation | Time spent in operations |
//! | `qmdb_operations_total` | Counter | operation | Total operation count |
//!
//! ## I/O Metrics
//!
//! | Metric | Type | Labels | Description |
//! |--------|------|--------|-------------|
//! | `qmdb_bytes_total` | Counter | direction | Bytes read/written |
//! | `qmdb_io_operations_total` | Counter | operation | I/O operation count |
//!
//! ## Cache Metrics
//!
//! | Metric | Type | Description |
//! |--------|------|-------------|
//! | `qmdb_cache_hits_total` | Counter | Cache hit count |
//! | `qmdb_cache_misses_total` | Counter | Cache miss count |
//! | `qmdb_cache_size_bytes` | Gauge | Current cache size |
//!
//! ## Tree Metrics
//!
//! | Metric | Type | Description |
//! |--------|------|-------------|
//! | `qmdb_mmr_size` | Gauge | MMR node count |
//! | `qmdb_bitmap_size_bits` | Gauge | Bitmap bit count |
//! | `qmdb_twig_count` | Gauge | Twig count (cyclone-compat) |
```

## Files to Create

- `storage/src/metrics.rs`

## Files to Modify

- `storage/src/lib.rs` (add metrics module)
- `storage/src/qmdb/*.rs` (add instrumentation)
- `storage/src/mmr/*.rs` (add instrumentation)
- `storage/src/bitmap/*.rs` (add instrumentation)
- `storage/Cargo.toml` (add prometheus dependency)

## Testing Strategy

- Unit tests for metric helpers
- Integration test verifying metrics are recorded
- Validate metric names follow Prometheus conventions

## Dependencies

- None (can be done in parallel)

## Estimated Effort

- 1 day for audit and metric definition
- 1-2 days for instrumentation
- 0.5 day for documentation
