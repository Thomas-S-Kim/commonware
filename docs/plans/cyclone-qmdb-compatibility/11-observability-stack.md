# Task 11: Integrate Observability Stack

## Current State

- No integrated observability for benchmarks
- Manual profiling required to identify hotspots
- No system-level metrics correlation
- No pre-built dashboards for analysis

## Expected Goal

Full observability stack for performance analysis including:
- Grafana Pyroscope for continuous profiling
- Prometheus Node Exporter for system metrics
- Grafana dashboards for visualization
- Docker Compose setup for local development

## Rationale

Integrated observability enables:
1. **Profiling without instrumentation**: Pyroscope identifies hotspots automatically
2. **System correlation**: Correlate DB performance with CPU/memory/disk
3. **Visualization**: Pre-built dashboards for common analyses
4. **Reproducibility**: Same stack locally and in CI

## Execution Plan

### Step 1: Create Docker Compose Setup

**File**: `tools/qmdb-bench/docker-compose.yml`

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: qmdb-prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
    networks:
      - qmdb-bench

  pushgateway:
    image: prom/pushgateway:latest
    container_name: qmdb-pushgateway
    ports:
      - "9091:9091"
    networks:
      - qmdb-bench

  node-exporter:
    image: prom/node-exporter:latest
    container_name: qmdb-node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.ignored-mount-points=^/(sys|proc|dev|host|etc)($$|/)'
    networks:
      - qmdb-bench

  pyroscope:
    image: grafana/pyroscope:latest
    container_name: qmdb-pyroscope
    ports:
      - "4040:4040"
    volumes:
      - pyroscope-data:/var/lib/pyroscope
    networks:
      - qmdb-bench

  grafana:
    image: grafana/grafana:latest
    container_name: qmdb-grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards
      - grafana-data:/var/lib/grafana
    depends_on:
      - prometheus
      - pyroscope
    networks:
      - qmdb-bench

networks:
  qmdb-bench:
    driver: bridge

volumes:
  prometheus-data:
  pyroscope-data:
  grafana-data:
```

### Step 2: Configure Prometheus

**File**: `tools/qmdb-bench/prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  # Scrape pushgateway for benchmark results
  - job_name: 'pushgateway'
    honor_labels: true
    static_configs:
      - targets: ['pushgateway:9091']

  # Scrape node exporter for system metrics
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']

  # Scrape Prometheus itself
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

### Step 3: Configure Grafana Data Sources

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

### Step 4: Create Grafana Dashboards

**File**: `tools/qmdb-bench/grafana/dashboards/qmdb-overview.json`

```json
{
  "title": "QMDB Benchmark Overview",
  "uid": "qmdb-overview",
  "panels": [
    {
      "title": "Operations per Second",
      "type": "timeseries",
      "targets": [
        {
          "expr": "qmdb_bench_ops_per_sec",
          "legendFormat": "{{workload}} - {{mode}}"
        }
      ],
      "gridPos": { "x": 0, "y": 0, "w": 12, "h": 8 }
    },
    {
      "title": "Latency Percentiles",
      "type": "timeseries",
      "targets": [
        {
          "expr": "qmdb_bench_latency_seconds{percentile='p50'}",
          "legendFormat": "p50 - {{workload}}"
        },
        {
          "expr": "qmdb_bench_latency_seconds{percentile='p99'}",
          "legendFormat": "p99 - {{workload}}"
        }
      ],
      "gridPos": { "x": 12, "y": 0, "w": 12, "h": 8 }
    },
    {
      "title": "CPU Usage",
      "type": "timeseries",
      "targets": [
        {
          "expr": "100 - (avg(irate(node_cpu_seconds_total{mode='idle'}[5m])) * 100)",
          "legendFormat": "CPU %"
        }
      ],
      "gridPos": { "x": 0, "y": 8, "w": 8, "h": 6 }
    },
    {
      "title": "Memory Usage",
      "type": "timeseries",
      "targets": [
        {
          "expr": "node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes",
          "legendFormat": "Used Memory"
        }
      ],
      "gridPos": { "x": 8, "y": 8, "w": 8, "h": 6 }
    },
    {
      "title": "Disk I/O",
      "type": "timeseries",
      "targets": [
        {
          "expr": "rate(node_disk_read_bytes_total[5m])",
          "legendFormat": "Read"
        },
        {
          "expr": "rate(node_disk_written_bytes_total[5m])",
          "legendFormat": "Write"
        }
      ],
      "gridPos": { "x": 16, "y": 8, "w": 8, "h": 6 }
    }
  ]
}
```

**File**: `tools/qmdb-bench/grafana/dashboards/qmdb-comparison.json`

```json
{
  "title": "Native vs Cyclone Comparison",
  "uid": "qmdb-comparison",
  "panels": [
    {
      "title": "Throughput Comparison",
      "type": "bargauge",
      "targets": [
        {
          "expr": "qmdb_bench_ops_per_sec{mode='native'}",
          "legendFormat": "Native"
        },
        {
          "expr": "qmdb_bench_ops_per_sec{mode='cyclone'}",
          "legendFormat": "Cyclone"
        }
      ]
    },
    {
      "title": "Latency Comparison (p99)",
      "type": "bargauge",
      "targets": [
        {
          "expr": "qmdb_bench_latency_seconds{mode='native', percentile='p99'}",
          "legendFormat": "Native"
        },
        {
          "expr": "qmdb_bench_latency_seconds{mode='cyclone', percentile='p99'}",
          "legendFormat": "Cyclone"
        }
      ]
    }
  ]
}
```

### Step 5: Add Pyroscope Integration to Benchmark CLI

**File**: `tools/qmdb-bench/src/profiling.rs`

```rust
//! Pyroscope continuous profiling integration.

use pyroscope::PyroscopeAgent;
use pyroscope_pprofrs::{pprof_backend, PprofConfig};

pub struct Profiler {
    agent: Option<PyroscopeAgent<pprof_backend::PprofBackend>>,
}

impl Profiler {
    pub fn new(pyroscope_url: Option<&str>, app_name: &str) -> anyhow::Result<Self> {
        let agent = if let Some(url) = pyroscope_url {
            let config = PprofConfig::new().sample_rate(100);
            let agent = PyroscopeAgent::builder(url, app_name)
                .backend(pprof_backend::PprofBackend::new(config))
                .build()?;
            Some(agent)
        } else {
            None
        };

        Ok(Self { agent })
    }

    pub fn start(&mut self) -> anyhow::Result<()> {
        if let Some(ref mut agent) = self.agent {
            agent.start()?;
        }
        Ok(())
    }

    pub fn add_tag(&mut self, key: &str, value: &str) {
        if let Some(ref mut agent) = self.agent {
            agent.add_global_tag(key.into(), value.into());
        }
    }

    pub fn stop(&mut self) -> anyhow::Result<()> {
        if let Some(ref mut agent) = self.agent {
            agent.stop()?;
        }
        Ok(())
    }
}
```

### Step 6: Create Helper Scripts

**File**: `tools/qmdb-bench/scripts/start-observability.sh`

```bash
#!/bin/bash
set -e

cd "$(dirname "$0")/.."

echo "Starting observability stack..."
docker-compose up -d

echo "Waiting for services to be ready..."
sleep 5

echo ""
echo "Services available at:"
echo "  Grafana:     http://localhost:3000 (admin/admin)"
echo "  Prometheus:  http://localhost:9090"
echo "  Pushgateway: http://localhost:9091"
echo "  Pyroscope:   http://localhost:4040"
echo ""
echo "To run benchmarks with profiling:"
echo "  qmdb-bench --workload configs/mixed_80_20.toml \\"
echo "             --output prometheus \\"
echo "             --push-gateway http://localhost:9091 \\"
echo "             --pyroscope http://localhost:4040"
```

**File**: `tools/qmdb-bench/scripts/stop-observability.sh`

```bash
#!/bin/bash
cd "$(dirname "$0")/.."
docker-compose down
```

## Files to Create

- `tools/qmdb-bench/docker-compose.yml`
- `tools/qmdb-bench/prometheus/prometheus.yml`
- `tools/qmdb-bench/grafana/provisioning/datasources/datasources.yml`
- `tools/qmdb-bench/grafana/provisioning/dashboards/dashboards.yml`
- `tools/qmdb-bench/grafana/dashboards/*.json`
- `tools/qmdb-bench/src/profiling.rs`
- `tools/qmdb-bench/scripts/*.sh`

## Files to Modify

- `tools/qmdb-bench/Cargo.toml` (add pyroscope dependencies)
- `tools/qmdb-bench/src/main.rs` (integrate profiling)

## Testing Strategy

- Verify Docker Compose stack starts correctly
- Test Prometheus scraping from pushgateway
- Validate Grafana dashboards load
- Test Pyroscope profiling captures data

## Dependencies

- Task 9 (Benchmark CLI)
- Task 10 (Storage telemetry)

## Estimated Effort

- 1 day for Docker Compose and config
- 1 day for Grafana dashboards
- 0.5 day for Pyroscope integration
- 0.5 day for testing
