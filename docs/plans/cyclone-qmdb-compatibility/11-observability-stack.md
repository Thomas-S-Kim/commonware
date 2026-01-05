# Task 11: Observability Stack

## Current State

- No integrated observability for benchmarks
- Manual profiling required

## Goal

Docker Compose stack with Prometheus, Grafana, Pyroscope for local development.

## Execution

### Step 1: Docker Compose

**File**: `tools/qmdb-bench/docker-compose.yml`

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    networks:
      - qmdb-bench

  pushgateway:
    image: prom/pushgateway:latest
    ports:
      - "9091:9091"
    networks:
      - qmdb-bench

  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
    networks:
      - qmdb-bench

  pyroscope:
    image: grafana/pyroscope:latest
    ports:
      - "4040:4040"
    networks:
      - qmdb-bench

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards
    depends_on:
      - prometheus
      - pyroscope
    networks:
      - qmdb-bench

networks:
  qmdb-bench:
    driver: bridge
```

### Step 2: Prometheus Config

**File**: `tools/qmdb-bench/prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'pushgateway'
    honor_labels: true
    static_configs:
      - targets: ['pushgateway:9091']

  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
```

### Step 3: Grafana Datasources

**File**: `tools/qmdb-bench/grafana/provisioning/datasources/datasources.yml`

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true

  - name: Pyroscope
    type: grafana-pyroscope-datasource
    access: proxy
    url: http://pyroscope:4040
```

### Step 4: Dashboard

**File**: `tools/qmdb-bench/grafana/dashboards/qmdb-overview.json`

```json
{
  "title": "QMDB Benchmark Overview",
  "uid": "qmdb-overview",
  "panels": [
    {
      "title": "Operations per Second",
      "type": "timeseries",
      "targets": [{"expr": "qmdb_bench_ops_per_sec", "legendFormat": "{{workload}} - {{scheme}}"}],
      "gridPos": {"x": 0, "y": 0, "w": 12, "h": 8}
    },
    {
      "title": "Latency Percentiles",
      "type": "timeseries",
      "targets": [
        {"expr": "qmdb_bench_latency_seconds{percentile='p50'}", "legendFormat": "p50"},
        {"expr": "qmdb_bench_latency_seconds{percentile='p99'}", "legendFormat": "p99"}
      ],
      "gridPos": {"x": 12, "y": 0, "w": 12, "h": 8}
    },
    {
      "title": "CPU Usage",
      "type": "timeseries",
      "targets": [{"expr": "100 - (avg(irate(node_cpu_seconds_total{mode='idle'}[5m])) * 100)"}],
      "gridPos": {"x": 0, "y": 8, "w": 8, "h": 6}
    },
    {
      "title": "Disk I/O",
      "type": "timeseries",
      "targets": [
        {"expr": "rate(node_disk_read_bytes_total[5m])", "legendFormat": "Read"},
        {"expr": "rate(node_disk_written_bytes_total[5m])", "legendFormat": "Write"}
      ],
      "gridPos": {"x": 8, "y": 8, "w": 8, "h": 6}
    }
  ]
}
```

### Step 5: Pyroscope Integration

**File**: `tools/qmdb-bench/src/profiling.rs`

```rust
use pyroscope::PyroscopeAgent;
use pyroscope_pprofrs::{pprof_backend, PprofConfig};

pub struct Profiler {
    agent: Option<PyroscopeAgent<pprof_backend::PprofBackend>>,
}

impl Profiler {
    pub fn new(pyroscope_url: Option<&str>, app_name: &str) -> anyhow::Result<Self> {
        let agent = pyroscope_url.map(|url| {
            let config = PprofConfig::new().sample_rate(100);
            PyroscopeAgent::builder(url, app_name)
                .backend(pprof_backend::PprofBackend::new(config))
                .build()
        }).transpose()?;
        Ok(Self { agent })
    }

    pub fn start(&mut self) -> anyhow::Result<()> {
        if let Some(ref mut agent) = self.agent { agent.start()?; }
        Ok(())
    }

    pub fn stop(&mut self) -> anyhow::Result<()> {
        if let Some(ref mut agent) = self.agent { agent.stop()?; }
        Ok(())
    }
}
```

### Step 6: Helper Scripts

**File**: `tools/qmdb-bench/scripts/start-observability.sh`

```bash
#!/bin/bash
set -e
cd "$(dirname "$0")/.."
docker-compose up -d
echo "Grafana: http://localhost:3000 (admin/admin)"
echo "Prometheus: http://localhost:9090"
echo "Pyroscope: http://localhost:4040"
```

## Files to Create

- `tools/qmdb-bench/docker-compose.yml`
- `tools/qmdb-bench/prometheus/prometheus.yml`
- `tools/qmdb-bench/grafana/provisioning/datasources/datasources.yml`
- `tools/qmdb-bench/grafana/dashboards/*.json`
- `tools/qmdb-bench/src/profiling.rs`
- `tools/qmdb-bench/scripts/*.sh`

## Files to Modify

- `tools/qmdb-bench/Cargo.toml` (add pyroscope)
- `tools/qmdb-bench/src/main.rs` (integrate profiling)

## Testing

```bash
./scripts/start-observability.sh
qmdb-bench --workload configs/mixed_80_20.toml --output prometheus --push-gateway http://localhost:9091
# Check Grafana dashboard
./scripts/stop-observability.sh
```

## Dependencies

- Task 9 (Benchmark CLI)
- Task 10 (Storage telemetry)
