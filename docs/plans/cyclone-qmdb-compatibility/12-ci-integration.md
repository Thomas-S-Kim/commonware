# Task 12: CI Integration with Self-Hosted Runners

## Current State

- CI runs on GitHub-hosted runners
- No automated performance tracking
- No performance regression detection
- No historical performance data

## Expected Goal

Automated performance regression detection in CI with:
- Self-hosted GitHub runners on consistent hardware
- Benchmark suite running on PRs and main branch
- Metrics pushed to long-lived Prometheus server
- Alerts on significant regressions

## Rationale

CI-integrated benchmarks provide:
1. **Regression detection**: Catch performance issues before merge
2. **Historical tracking**: Trend analysis over time
3. **Consistency**: Same hardware for all measurements
4. **Automation**: No manual benchmark running needed

## Execution Plan

### Step 1: Set Up Self-Hosted Runners

**Option A: AWS EC2 Instances**

Use `commonware-deployer` patterns to provision dedicated benchmark runners:

```hcl
# Example Terraform for benchmark runner
resource "aws_instance" "benchmark_runner" {
  ami           = "ami-xxx"  # Ubuntu 22.04
  instance_type = "c6i.2xlarge"  # 8 vCPU, 16GB RAM

  tags = {
    Name = "qmdb-benchmark-runner"
    Role = "github-runner"
  }

  user_data = <<-EOF
    #!/bin/bash
    # Install GitHub Actions runner
    # Install Rust toolchain
    # Install io_uring dependencies
    EOF
}
```

**Option B: Dedicated Long-Lived Machines**

Set up physical or cloud machines with:
- Fixed hardware configuration
- GitHub Actions runner installed
- Rust toolchain and dependencies
- io_uring support (Linux 5.10+)

### Step 2: Configure GitHub Actions Runner

**File**: `tools/qmdb-bench/ci/setup-runner.sh`

```bash
#!/bin/bash
set -e

# Install dependencies
sudo apt-get update
sudo apt-get install -y \
    build-essential \
    curl \
    git \
    linux-tools-generic \
    liburing-dev

# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source $HOME/.cargo/env

# Install GitHub Actions runner
mkdir -p /opt/actions-runner && cd /opt/actions-runner
curl -o actions-runner-linux-x64.tar.gz -L \
    https://github.com/actions/runner/releases/download/v2.xxx/actions-runner-linux-x64-2.xxx.tar.gz
tar xzf ./actions-runner-linux-x64.tar.gz

# Configure runner (requires GITHUB_TOKEN)
./config.sh --url https://github.com/commonwarexyz/commonware \
            --token $GITHUB_TOKEN \
            --labels benchmark,self-hosted \
            --unattended

# Install and start as service
sudo ./svc.sh install
sudo ./svc.sh start
```

### Step 3: Create Benchmark Workflow

**File**: `.github/workflows/benchmark.yml`

```yaml
name: Performance Benchmarks

on:
  push:
    branches: [main]
    paths:
      - 'storage/**'
      - 'tools/qmdb-bench/**'
  pull_request:
    paths:
      - 'storage/**'
      - 'tools/qmdb-bench/**'
  workflow_dispatch:
    inputs:
      workload:
        description: 'Workload config to run'
        required: false
        default: 'full_suite'

env:
  PROMETHEUS_PUSHGATEWAY: ${{ secrets.PROMETHEUS_PUSHGATEWAY }}
  CARGO_TERM_COLOR: always

jobs:
  benchmark:
    name: Run Benchmarks
    runs-on: [self-hosted, benchmark]
    timeout-minutes: 60

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Rust
        uses: dtolnay/rust-action@stable

      - name: Build benchmark tool
        run: |
          cargo build --release -p qmdb-bench

      - name: Run benchmarks
        id: benchmark
        run: |
          WORKLOAD="${{ github.event.inputs.workload || 'full_suite' }}"
          ./target/release/qmdb-bench \
            --workload tools/qmdb-bench/configs/${WORKLOAD}.toml \
            --runtime iouring \
            --output json \
            --iterations 5 \
            > benchmark-results.json

          # Extract key metrics for summary
          OPS=$(jq '.ops_per_sec.mean' benchmark-results.json)
          P99=$(jq '.latencies.p99_ms' benchmark-results.json)
          echo "ops_per_sec=${OPS}" >> $GITHUB_OUTPUT
          echo "p99_latency=${P99}" >> $GITHUB_OUTPUT

      - name: Push to Prometheus
        if: github.ref == 'refs/heads/main'
        run: |
          ./target/release/qmdb-bench \
            --workload tools/qmdb-bench/configs/full_suite.toml \
            --runtime iouring \
            --output prometheus \
            --push-gateway $PROMETHEUS_PUSHGATEWAY \
            --job-name "qmdb_bench_main"

      - name: Check for regression
        if: github.event_name == 'pull_request'
        run: |
          ./tools/qmdb-bench/ci/check-regression.sh \
            benchmark-results.json \
            ${{ secrets.PROMETHEUS_URL }}

      - name: Comment on PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const ops = '${{ steps.benchmark.outputs.ops_per_sec }}';
            const p99 = '${{ steps.benchmark.outputs.p99_latency }}';
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Benchmark Results\n\n| Metric | Value |\n|--------|-------|\n| Ops/sec | ${ops} |\n| P99 Latency | ${p99}ms |`
            });

      - name: Upload results
        uses: actions/upload-artifact@v4
        with:
          name: benchmark-results
          path: benchmark-results.json
```

### Step 4: Create Regression Check Script

**File**: `tools/qmdb-bench/ci/check-regression.sh`

```bash
#!/bin/bash
set -e

RESULTS_FILE=$1
PROMETHEUS_URL=$2
THRESHOLD=${3:-10}  # 10% regression threshold

# Get current results
CURRENT_OPS=$(jq '.ops_per_sec.mean' "$RESULTS_FILE")
CURRENT_P99=$(jq '.latencies.p99_ms' "$RESULTS_FILE")

# Query baseline from Prometheus (last 7 days average on main)
BASELINE_OPS=$(curl -s "${PROMETHEUS_URL}/api/v1/query" \
  --data-urlencode "query=avg_over_time(qmdb_bench_ops_per_sec{job='qmdb_bench_main'}[7d])" \
  | jq -r '.data.result[0].value[1] // 0')

BASELINE_P99=$(curl -s "${PROMETHEUS_URL}/api/v1/query" \
  --data-urlencode "query=avg_over_time(qmdb_bench_latency_seconds{job='qmdb_bench_main',percentile='p99'}[7d])" \
  | jq -r '.data.result[0].value[1] // 0')

# Skip if no baseline
if [ "$BASELINE_OPS" == "0" ] || [ "$BASELINE_OPS" == "null" ]; then
  echo "No baseline data available, skipping regression check"
  exit 0
fi

# Calculate regression percentage
OPS_DIFF=$(echo "scale=2; (($BASELINE_OPS - $CURRENT_OPS) / $BASELINE_OPS) * 100" | bc)
P99_DIFF=$(echo "scale=2; (($CURRENT_P99 - $BASELINE_P99) / $BASELINE_P99) * 100" | bc)

echo "Throughput change: ${OPS_DIFF}% (baseline: ${BASELINE_OPS}, current: ${CURRENT_OPS})"
echo "P99 latency change: ${P99_DIFF}% (baseline: ${BASELINE_P99}ms, current: ${CURRENT_P99}ms)"

# Check if regression exceeds threshold
if (( $(echo "$OPS_DIFF > $THRESHOLD" | bc -l) )); then
  echo "ERROR: Throughput regression of ${OPS_DIFF}% exceeds threshold of ${THRESHOLD}%"
  exit 1
fi

if (( $(echo "$P99_DIFF > $THRESHOLD" | bc -l) )); then
  echo "ERROR: P99 latency regression of ${P99_DIFF}% exceeds threshold of ${THRESHOLD}%"
  exit 1
fi

echo "No significant regression detected"
```

### Step 5: Set Up Long-Lived Prometheus

For historical data, set up a long-lived Prometheus instance:

**Option A: Managed Service**
- Grafana Cloud
- AWS Managed Prometheus
- Google Cloud Managed Prometheus

**Option B: Self-Hosted**
- EC2 instance with persistent storage
- Use Thanos or Cortex for long-term storage

**Required Configuration:**
- Retention: 90+ days
- Remote write from benchmark runners
- Alerting rules for regressions

### Step 6: Configure Alerts

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
          summary: "QMDB throughput regression detected"
          description: "Throughput has dropped by more than 10% compared to last week"

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
          summary: "QMDB latency regression detected"
          description: "P99 latency has increased by more than 10% compared to last week"
```

## Files to Create

- `.github/workflows/benchmark.yml`
- `tools/qmdb-bench/ci/setup-runner.sh`
- `tools/qmdb-bench/ci/check-regression.sh`
- `tools/qmdb-bench/prometheus/alerts.yml`

## GitHub Secrets Required

| Secret | Description |
|--------|-------------|
| `PROMETHEUS_PUSHGATEWAY` | URL of Prometheus pushgateway |
| `PROMETHEUS_URL` | URL of Prometheus for baseline queries |
| `GITHUB_TOKEN` | For runner registration (use default) |

## Platform Considerations

| Platform | Support Level |
|----------|---------------|
| Linux (io_uring) | Full support, recommended for benchmarks |
| Linux (standard) | Full support |
| macOS | Limited, for development only |
| Windows | Not recommended for benchmarks |

## Testing Strategy

- Test workflow on a branch before merging
- Verify runner connectivity and authentication
- Test regression detection with synthetic data
- Validate alert firing

## Dependencies

- Task 9 (Benchmark CLI)
- Task 11 (Observability stack for local testing)

## Estimated Effort

- 1 day for runner setup
- 1 day for workflow implementation
- 0.5 day for regression detection
- 0.5 day for alerts and monitoring
