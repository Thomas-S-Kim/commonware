# Cyclone QMDB Compatibility - Overview

This directory contains the implementation plan for adding cyclone QMDB state root compatibility to commonware-storage, along with automated performance testing infrastructure.

## Background

### Problem Statement

Commonware-storage and cyclone QMDB both implement authenticated key-value databases but use different internal structures and hashing schemes. To enable interoperability, we need commonware to optionally produce identical state roots to cyclone for the same input data.

### Key Differences

| Aspect | Commonware | Cyclone |
|--------|------------|---------|
| Hash function | Position-based, configurable | Blake3 keyed with level |
| Active bits tree | MMR-like, configurable N | Fixed 3-level binary tree |
| Entries per chunk | Configurable via grafting | Fixed 2048 per twig |
| Chunk/page size | N bytes (generic) | Fixed 32 bytes |
| Domain separation | Position in hash input | Level XOR'd into Blake3 IV |

### Design Principles

1. **Preserve existing behavior**: All current APIs and implementations remain unchanged
2. **Add new abstractions**: Cyclone compatibility via opt-in traits and types
3. **Feature-flagged**: New code gated behind `cyclone-compat` feature
4. **Test-driven**: Validate against cyclone-generated test vectors

## Task Execution Order

Reorder the tasks below based on priority before execution. Each task corresponds to one PR.

```
# EDIT THIS LIST TO CHANGE EXECUTION ORDER
# Format: [status] task-file.md - Brief description

[ ] 01-generate-test-vectors.md     - Generate test vectors from cyclone (ground truth)
[ ] 02-blake3-hasher.md             - Add Blake3 level-keyed hasher
[ ] 03-cyclone-twig.md              - Add CycloneTwig structure
[ ] 04-cyclone-root-calculation.md  - Add cyclone-compatible root calculation
[ ] 05-compatibility-trait.md       - Create CycloneCompatible trait
[ ] 06-compatibility-tests.md       - Write compatibility tests
[ ] 07-documentation-feature-flag.md - Documentation and feature flag
[ ] 08-define-workloads.md          - Define QMDB benchmark workloads
[ ] 09-benchmark-cli.md             - Build benchmark CLI tool
[ ] 10-storage-telemetry.md         - Audit and extend storage telemetry
[ ] 11-observability-stack.md       - Integrate observability stack
[ ] 12-ci-integration.md            - CI integration with self-hosted runners
```

### Dependency Graph

```
                    +-------------------+
                    | 01-test-vectors   |
                    +-------------------+
                            |
                            v
+-------------------+       |
| 02-blake3-hasher  |-------+
+-------------------+       |
        |                   |
        v                   |
+-------------------+       |
| 03-cyclone-twig   |       |
+-------------------+       |
        |                   |
        v                   |
+-------------------+       |
| 04-root-calc      |       |
+-------------------+       |
        |                   |
        v                   |
+-------------------+       |
| 05-compat-trait   |       |
+-------------------+       |
        |                   |
        +-------+-----------+
                |
                v
+-------------------+
| 06-compat-tests   |
+-------------------+
        |
        v
+-------------------+
| 07-docs-feature   |
+-------------------+


+-------------------+     +-------------------+
| 08-workloads      |     | 10-telemetry      |
+-------------------+     +-------------------+
        |                         |
        v                         |
+-------------------+             |
| 09-bench-cli      |<------------+
+-------------------+
        |
        v
+-------------------+
| 11-observability  |
+-------------------+
        |
        v
+-------------------+
| 12-ci-integration |
+-------------------+
```

### Suggested Phases

**Phase 1: Core Compatibility (Tasks 1-7)**
- Can be done first to establish cyclone state root equivalence
- Estimated: 4-6 PRs

**Phase 2: Performance Infrastructure (Tasks 8-12)**
- Can be done in parallel or after Phase 1
- Estimated: 3-5 PRs

## References

### Codebases
- Cyclone QMDB: `/Users/thomas/projects/thomas/cyclone/crates/qmdb`
- Commonware Storage: `/Users/thomas/projects/commonware/storage`
- Commonware Deployer: `/Users/thomas/projects/commonware/deployer`

### Key Cyclone Files
- `qmdb-common/src/merkletree/hash.rs` - `merkle_node_hash` with level-keyed Blake3
- `qmdb-common/src/merkletree/twig.rs` - Twig structure with `sync_l1/l2/l3/top`
- `qmdb-common/src/utils/hasher.rs` - Blake3 keyed hash implementation
- `qmdb-common/src/def.rs` - Constants (TWIG_SHIFT=11, LEAF_COUNT_IN_TWIG=2048)

### Key Commonware Files
- `storage/src/mmr/hasher.rs` - Hasher trait and Standard implementation
- `storage/src/mmr/grafting.rs` - Grafting hasher and storage
- `storage/src/bitmap/authenticated.rs` - Authenticated bitmap with MMR-like structure
- `storage/src/qmdb/current/mod.rs` - Current database with grafted bitmap

### Design Documents
- [Performance comparison](https://gist.github.com/absolute0kelvin/effcae915772881cea6e81a51c602b72)
- [Compatibility notes](https://gist.github.com/absolute0kelvin/82361399310ea4a349220952df7ea256)

### Observability Tools
- [Grafana Pyroscope](https://grafana.com/docs/pyroscope/latest/) - Continuous profiling
- [Prometheus Node Exporter](https://prometheus.io/docs/guides/node-exporter/) - System metrics

## Open Questions

### Compatibility
1. **Feature flag naming**: `cyclone-compat` or more generic `external-compat`?
2. **Test vector generation**: Binary in cyclone repo or use existing test infra?
3. **Proof verification**: Include in Task 6 or defer to follow-up PR?

### Performance Testing
4. **Benchmark tool location**: `storage/src/bin/`, separate `tools/` crate, or extend example?
5. **Self-hosted runner provisioning**: Use `commonware-deployer` or dedicated instances?
6. **Metrics retention**: 30 days? 90 days?
7. **Regression threshold**: 5%? 10%?
8. **Workload prioritization**: Which workloads first? (Storage team input needed)
9. **Telemetry gaps**: Need audit of existing metrics

## Success Criteria

- [ ] Commonware computes identical state roots to cyclone for same input
- [ ] All existing tests pass without modification
- [ ] Compatibility tests validate state root equivalence
- [ ] No breaking changes to existing public APIs
- [ ] Automated benchmarks run in CI with regression detection
- [ ] Performance dashboards available for analysis
