# Task 12: CI Integration

## Current State

- CI runs on GitHub-hosted runners
- No automated performance tracking
- No regression detection

## Goal

Automated performance regression detection with self-hosted runners and Prometheus metrics.

## Execution

### Step 1: Self-Hosted Runner Setup

**File**: `tools/qmdb-bench/ci/setup-runner.sh`

```bash
#!/bin/bash
set -e

sudo apt-get update
sudo apt-get install -y build-essential curl git linux-tools-generic liburing-dev

curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source $HOME/.cargo/env

mkdir -p /opt/actions-runner && cd /opt/actions-runner
curl -o actions-runner-linux-x64.tar.gz -L \
    https://github.com/actions/runner/releases/download/v2.xxx/actions-runner-linux-x64-2.xxx.tar.gz
tar xzf ./actions-runner-linux-x64.tar.gz

./config.sh --url https://github.com/commonwarexyz/commonware \
            --token $GITHUB_TOKEN \
            --labels benchmark,self-hosted \
            --unattended

sudo ./svc.sh install
sudo ./svc.sh start
```

### Step 2: Benchmark Workflow

**File**: `.github/workflows/benchmark.yml`

```yaml
name: Performance Benchmarks

on:
  push:
    branches: [main]
    paths: ['storage/**', 'tools/qmdb-bench/**']
  pull_request:
    paths: ['storage/**', 'tools/qmdb-bench/**']
  workflow_dispatch:

env:
  PROMETHEUS_PUSHGATEWAY: ${{ secrets.PROMETHEUS_PUSHGATEWAY }}

jobs:
  benchmark:
    runs-on: [self-hosted, benchmark]
    timeout-minutes: 60

    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-action@stable

      - name: Build
        run: cargo build --release -p qmdb-bench

      - name: Run benchmarks
        id: benchmark
        run: |
          ./target/release/qmdb-bench \
            --workload tools/qmdb-bench/configs/full_suite.toml \
            --runtime iouring \
            --output json \
            --iterations 5 \
            > benchmark-results.json
          echo "ops_per_sec=$(jq '.ops_per_sec.mean' benchmark-results.json)" >> $GITHUB_OUTPUT
          echo "p99_latency=$(jq '.latencies.p99_ms' benchmark-results.json)" >> $GITHUB_OUTPUT

      - name: Push to Prometheus
        if: github.ref == 'refs/heads/main'
        run: |
          ./target/release/qmdb-bench \
            --workload tools/qmdb-bench/configs/full_suite.toml \
            --runtime iouring \
            --output prometheus \
            --push-gateway $PROMETHEUS_PUSHGATEWAY

      - name: Check regression
        if: github.event_name == 'pull_request'
        run: ./tools/qmdb-bench/ci/check-regression.sh benchmark-results.json ${{ secrets.PROMETHEUS_URL }}

      - name: Comment on PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Benchmark Results\n\n| Metric | Value |\n|--------|-------|\n| Ops/sec | ${{ steps.benchmark.outputs.ops_per_sec }} |\n| P99 Latency | ${{ steps.benchmark.outputs.p99_latency }}ms |`
            });

      - uses: actions/upload-artifact@v4
        with:
          name: benchmark-results
          path: benchmark-results.json
```

### Step 3: Regression Check Script

**File**: `tools/qmdb-bench/ci/check-regression.sh`

```bash
#!/bin/bash
set -e

RESULTS_FILE=$1
PROMETHEUS_URL=$2
THRESHOLD=${3:-10}

CURRENT_OPS=$(jq '.ops_per_sec.mean' "$RESULTS_FILE")
CURRENT_P99=$(jq '.latencies.p99_ms' "$RESULTS_FILE")

BASELINE_OPS=$(curl -s "${PROMETHEUS_URL}/api/v1/query" \
  --data-urlencode "query=avg_over_time(qmdb_bench_ops_per_sec{job='qmdb_bench_main'}[7d])" \
  | jq -r '.data.result[0].value[1] // 0')

if [ "$BASELINE_OPS" == "0" ] || [ "$BASELINE_OPS" == "null" ]; then
  echo "No baseline, skipping"
  exit 0
fi

OPS_DIFF=$(echo "scale=2; (($BASELINE_OPS - $CURRENT_OPS) / $BASELINE_OPS) * 100" | bc)

echo "Throughput change: ${OPS_DIFF}% (baseline: ${BASELINE_OPS}, current: ${CURRENT_OPS})"

if (( $(echo "$OPS_DIFF > $THRESHOLD" | bc -l) )); then
  echo "ERROR: Throughput regression of ${OPS_DIFF}% exceeds ${THRESHOLD}%"
  exit 1
fi

echo "No significant regression"
```

### Step 4: Prometheus Alerts

**File**: `tools/qmdb-bench/prometheus/alerts.yml`

```yaml
groups:
  - name: qmdb-benchmarks
    rules:
      - alert: ThroughputRegression
        expr: |
          (
            avg_over_time(qmdb_bench_ops_per_sec{job="qmdb_bench_main"}[1d])
            /
            avg_over_time(qmdb_bench_ops_per_sec{job="qmdb_bench_main"}[7d] offset 1d)
          ) < 0.9
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "QMDB throughput dropped >10%"

      - alert: LatencyRegression
        expr: |
          (
            avg_over_time(qmdb_bench_latency_seconds{percentile="p99"}[1d])
            /
            avg_over_time(qmdb_bench_latency_seconds{percentile="p99"}[7d] offset 1d)
          ) > 1.1
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "QMDB p99 latency increased >10%"
```

## Files to Create

- `.github/workflows/benchmark.yml`
- `tools/qmdb-bench/ci/setup-runner.sh`
- `tools/qmdb-bench/ci/check-regression.sh`
- `tools/qmdb-bench/prometheus/alerts.yml`

## GitHub Secrets Required

| Secret | Description |
|--------|-------------|
| `PROMETHEUS_PUSHGATEWAY` | Pushgateway URL |
| `PROMETHEUS_URL` | Prometheus URL for baseline queries |

## Platform Support

| Platform | Support |
|----------|---------|
| Linux (io_uring) | Full, recommended |
| Linux (standard) | Full |
| macOS | Development only |

## Testing

- Test workflow on branch before merge
- Verify runner connectivity
- Test regression detection with synthetic data

## Dependencies

- Task 9 (Benchmark CLI)
- Task 11 (Observability stack)
