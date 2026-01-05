# Task 10: Storage Telemetry

## Current State

- Incomplete telemetry coverage
- No comprehensive metric documentation

## Goal

Prometheus-compatible metrics for all key operations.

## Execution

### Step 1: Audit Existing Metrics

Review:
- `storage/src/mmr/*.rs`
- `storage/src/bitmap/*.rs`
- `storage/src/qmdb/*.rs`
- `storage/src/journal/*.rs`

### Step 2: Required Metrics

**Operations:**
```
qmdb_operation_duration_seconds{operation="add|get|proof|commit"}
qmdb_operations_total{operation="add|get|proof|commit"}
qmdb_bytes_total{direction="read|write"}
```

**Cache:**
```
qmdb_cache_hits_total
qmdb_cache_misses_total
qmdb_cache_size_bytes
```

**Tree:**
```
qmdb_mmr_size
qmdb_bitmap_size_bits
qmdb_segment_count (balanced_tree)
```

**I/O:**
```
qmdb_io_operations_total{operation="read|write|sync"}
qmdb_io_bytes_total{direction="read|write"}
```

### Step 3: Implementation

**File**: `storage/src/metrics.rs`

```rust
use prometheus::{
    register_counter_vec, register_gauge, register_histogram_vec,
    CounterVec, Gauge, HistogramVec,
};
use once_cell::sync::Lazy;

pub static OPERATION_DURATION: Lazy<HistogramVec> = Lazy::new(|| {
    register_histogram_vec!(
        "qmdb_operation_duration_seconds",
        "Duration of QMDB operations",
        &["operation"],
        vec![0.0001, 0.0005, 0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0]
    ).unwrap()
});

pub static OPERATIONS_TOTAL: Lazy<CounterVec> = Lazy::new(|| {
    register_counter_vec!("qmdb_operations_total", "Total operations", &["operation"]).unwrap()
});

pub static BYTES_TOTAL: Lazy<CounterVec> = Lazy::new(|| {
    register_counter_vec!("qmdb_bytes_total", "Bytes transferred", &["direction"]).unwrap()
});

pub static CACHE_HITS: Lazy<Counter> = Lazy::new(|| {
    register_counter!("qmdb_cache_hits_total", "Cache hits").unwrap()
});

pub static CACHE_MISSES: Lazy<Counter> = Lazy::new(|| {
    register_counter!("qmdb_cache_misses_total", "Cache misses").unwrap()
});

pub struct OperationTimer {
    operation: &'static str,
    start: std::time::Instant,
}

impl OperationTimer {
    pub fn new(operation: &'static str) -> Self {
        Self { operation, start: std::time::Instant::now() }
    }
}

impl Drop for OperationTimer {
    fn drop(&mut self) {
        OPERATION_DURATION.with_label_values(&[self.operation]).observe(self.start.elapsed().as_secs_f64());
        OPERATIONS_TOTAL.with_label_values(&[self.operation]).inc();
    }
}

#[macro_export]
macro_rules! time_operation {
    ($op:expr, $body:expr) => {{
        let _timer = $crate::metrics::OperationTimer::new($op);
        $body
    }};
}
```

### Step 4: Instrument Code

**File**: `storage/src/qmdb/current/mod.rs`

```rust
use crate::metrics::{time_operation, BYTES_TOTAL, CACHE_HITS, CACHE_MISSES};

impl<...> CurrentDb<...> {
    pub async fn add(&mut self, key: &[u8], value: &[u8]) -> Result<u64, Error> {
        time_operation!("add", {
            BYTES_TOTAL.with_label_values(&["write"]).inc_by((key.len() + value.len()) as f64);
            // ... existing implementation ...
        })
    }

    pub async fn get(&self, key: &[u8]) -> Result<Option<Vec<u8>>, Error> {
        time_operation!("get", {
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

## Files to Create

- `storage/src/metrics.rs`

## Files to Modify

- `storage/src/lib.rs`
- `storage/src/qmdb/*.rs`
- `storage/Cargo.toml` (add prometheus)

## Testing

```bash
cargo test -p commonware-storage metrics
```

## Dependencies

- None (parallel)
