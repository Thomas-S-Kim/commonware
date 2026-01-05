# Task 1: Generate Cyclone Test Vectors

## Current State

- No test vectors exist for cross-system validation
- Cyclone QMDB is the reference implementation for state root calculation
- No way to verify commonware produces identical outputs

## Expected Goal

A comprehensive set of test vectors generated from cyclone QMDB that serve as ground truth for validating commonware's cyclone-compatible implementation.

## Rationale

Test vectors from cyclone provide:
1. **Ground truth**: Cyclone is the reference implementation
2. **Deterministic verification**: Hash comparisons are exact
3. **Regression detection**: Any divergence is immediately caught
4. **Documentation**: Test vectors serve as specification

## Execution Plan

### Step 1: Create Test Vector Generator in Cyclone

Create a test harness in the cyclone repository that outputs deterministic test cases.

**File**: `cyclone/crates/qmdb/tests/generate_test_vectors.rs`

```rust
// Pseudocode structure
fn main() {
    generate_empty_twig_vector();
    generate_single_entry_vector();
    generate_full_twig_vector();
    generate_multi_twig_vector();
    generate_active_bit_flip_vectors();
    generate_boundary_condition_vectors();
}
```

### Step 2: Define Test Vector Format

Output format: TOML with input data and expected hashes at each level.

```toml
# Example: single_twig.toml

[metadata]
description = "Single twig with 100 entries"
cyclone_version = "0.1.0"
generated_at = "2025-01-05T00:00:00Z"

[[entries]]
key = "0x..."
value = "0x..."
serial_number = 0

[[entries]]
key = "0x..."
value = "0x..."
serial_number = 1

# ... more entries ...

[expected_hashes]
# Active bits tree (3 levels)
active_bits_mtl1_0 = "0x..."
active_bits_mtl1_1 = "0x..."
active_bits_mtl1_2 = "0x..."
active_bits_mtl1_3 = "0x..."
active_bits_mtl2_0 = "0x..."
active_bits_mtl2_1 = "0x..."
active_bits_mtl3 = "0x..."

# Entry tree
left_root = "0x..."

# Combined
twig_root = "0x..."

# If multi-twig, include upper tree hashes
# upper_level_13_0 = "0x..."
```

### Step 3: Generate Vector Categories

| Category | Description | Count |
|----------|-------------|-------|
| Empty | Empty twig, no entries | 1 |
| Single entry | One entry at various positions | 5 |
| Full twig | Exactly 2048 entries | 1 |
| Partial twig | Various fill levels (10, 100, 500, 1000, 2000) | 5 |
| Multi-twig | 2, 4, 8, 16 twigs | 4 |
| Active bit flips | Deactivate entries at various positions | 10 |
| Boundary | Edge cases (first/last entry, twig boundaries) | 5 |

### Step 4: Store Vectors in Commonware

Copy generated vectors to commonware repository.

**Location**: `storage/src/qmdb/tests/fixtures/cyclone/`

```
fixtures/cyclone/
├── empty_twig.toml
├── single_entry_pos0.toml
├── single_entry_pos1000.toml
├── single_entry_pos2047.toml
├── full_twig.toml
├── partial_100.toml
├── partial_1000.toml
├── multi_twig_2.toml
├── multi_twig_8.toml
├── deactivate_first.toml
├── deactivate_last.toml
└── ...
```

## Files to Create

**In cyclone repo:**
- `cyclone/crates/qmdb/tests/generate_test_vectors.rs`
- `cyclone/crates/qmdb/tests/test_vector_format.rs` (format helpers)

**In commonware repo:**
- `storage/src/qmdb/tests/fixtures/cyclone/*.toml` (test vector files)
- `storage/src/qmdb/tests/fixtures/mod.rs` (loader utilities)

## Testing Strategy

1. Run generator in cyclone, verify it produces valid TOML
2. Manually inspect a few vectors for correctness
3. Version the vectors with cyclone commit hash
4. Create a simple loader test in commonware that parses all vectors

## Dependencies

- None (this task can be done first)

## Estimated Effort

- 1-2 days for generator implementation
- 0.5 day for vector review and documentation
