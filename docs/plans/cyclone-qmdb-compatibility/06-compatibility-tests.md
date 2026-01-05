# Task 6: Write Compatibility Tests

## Current State

- No cross-system compatibility tests exist
- Test vectors from Task 1 are available but not consumed
- No automated verification of state root equivalence

## Expected Goal

Comprehensive test suite that loads cyclone test vectors and validates state root equivalence at every level of the tree structure.

## Rationale

Compatibility tests are critical for:
1. **Validation**: Prove the implementation is correct
2. **Regression detection**: Catch any future divergence
3. **Documentation**: Test cases serve as specification
4. **Confidence**: Enable safe refactoring

## Execution Plan

### Step 1: Create Test Vector Loader

**File**: `storage/src/qmdb/tests/fixtures/mod.rs`

```rust
//! Test vector loading utilities.

use serde::Deserialize;
use std::collections::HashMap;

/// Metadata about a test vector.
#[derive(Debug, Deserialize)]
pub struct VectorMetadata {
    pub description: String,
    pub cyclone_version: String,
    #[serde(default)]
    pub generated_at: Option<String>,
}

/// An entry in the test vector.
#[derive(Debug, Deserialize)]
pub struct VectorEntry {
    pub key: String,
    pub value: String,
    pub serial_number: u64,
    #[serde(default)]
    pub deactivated: bool,
}

/// Expected hashes at various levels.
#[derive(Debug, Deserialize)]
pub struct ExpectedHashes {
    // Active bits tree
    pub active_bits_mtl1_0: Option<String>,
    pub active_bits_mtl1_1: Option<String>,
    pub active_bits_mtl1_2: Option<String>,
    pub active_bits_mtl1_3: Option<String>,
    pub active_bits_mtl2_0: Option<String>,
    pub active_bits_mtl2_1: Option<String>,
    pub active_bits_mtl3: Option<String>,

    // Entry tree
    pub left_root: Option<String>,

    // Twig root
    pub twig_root: Option<String>,

    // Upper tree (for multi-twig vectors)
    #[serde(flatten)]
    pub upper_tree: HashMap<String, String>,

    // Final state root
    pub state_root: String,
}

/// A complete test vector.
#[derive(Debug, Deserialize)]
pub struct TestVector {
    pub metadata: VectorMetadata,
    pub entries: Vec<VectorEntry>,
    pub expected_hashes: ExpectedHashes,
}

impl TestVector {
    /// Load a test vector from a TOML file.
    pub fn load(path: &str) -> Result<Self, Box<dyn std::error::Error>> {
        let content = std::fs::read_to_string(path)?;
        let vector: TestVector = toml::from_str(&content)?;
        Ok(vector)
    }

    /// Load all test vectors from the fixtures directory.
    pub fn load_all() -> Vec<(String, Self)> {
        let fixtures_dir = concat!(
            env!("CARGO_MANIFEST_DIR"),
            "/src/qmdb/tests/fixtures/cyclone"
        );

        let mut vectors = Vec::new();
        if let Ok(entries) = std::fs::read_dir(fixtures_dir) {
            for entry in entries.flatten() {
                let path = entry.path();
                if path.extension().map_or(false, |e| e == "toml") {
                    if let Ok(vector) = Self::load(path.to_str().unwrap()) {
                        let name = path.file_stem().unwrap().to_str().unwrap().to_string();
                        vectors.push((name, vector));
                    }
                }
            }
        }
        vectors
    }
}

/// Parse a hex string to bytes.
pub fn parse_hex(s: &str) -> Vec<u8> {
    let s = s.strip_prefix("0x").unwrap_or(s);
    hex::decode(s).expect("invalid hex string")
}

/// Parse a hex string to Hash32.
pub fn parse_hash32(s: &str) -> [u8; 32] {
    parse_hex(s).try_into().expect("hash must be 32 bytes")
}
```

### Step 2: Create Compatibility Test Module

**File**: `storage/src/qmdb/tests/cyclone_compat_tests.rs`

```rust
//! Compatibility tests validating state root equivalence with cyclone.

use super::fixtures::{TestVector, parse_hash32};
use crate::qmdb::cyclone_compat::{CycloneDb, CycloneMode};
use crate::bitmap::cyclone_twig::Twig;
use crate::mmr::cyclone_hasher::Hash32;

/// Test that empty database matches cyclone.
#[test]
fn test_empty_db_root() {
    let vector = TestVector::load(concat!(
        env!("CARGO_MANIFEST_DIR"),
        "/src/qmdb/tests/fixtures/cyclone/empty_twig.toml"
    )).expect("failed to load vector");

    let mut db = CycloneDb::<CycloneMode>::new();
    let root = db.root();

    let expected = parse_hash32(&vector.expected_hashes.state_root);
    assert_eq!(root, expected, "empty root mismatch");
}

/// Test all loaded vectors.
#[test]
fn test_all_vectors() {
    let vectors = TestVector::load_all();
    assert!(!vectors.is_empty(), "no test vectors found");

    for (name, vector) in vectors {
        println!("Testing vector: {}", name);
        test_vector(&name, &vector);
    }
}

fn test_vector(name: &str, vector: &TestVector) {
    let mut db = CycloneDb::<CycloneMode>::new();

    // Replay all entries
    for entry in &vector.entries {
        let key_bytes = super::fixtures::parse_hex(&entry.key);
        let value_bytes = super::fixtures::parse_hex(&entry.value);

        // Compute entry hash (matches cyclone's entry hashing)
        let entry_hash = compute_entry_hash(&key_bytes, &value_bytes);

        let sn = db.add(entry_hash);
        assert_eq!(sn, entry.serial_number, "serial number mismatch in {}", name);

        if entry.deactivated {
            db.deactivate(sn);
        }
    }

    // Verify final state root
    let root = db.root();
    let expected = parse_hash32(&vector.expected_hashes.state_root);
    assert_eq!(root, expected, "state root mismatch in {}", name);

    // Verify intermediate hashes if provided
    verify_intermediate_hashes(name, &db, &vector.expected_hashes);
}

fn compute_entry_hash(key: &[u8], value: &[u8]) -> Hash32 {
    // Match cyclone's entry hashing
    use crate::mmr::cyclone_hasher::CycloneMerkleHasher;
    let hasher = crate::mmr::cyclone_hasher::CycloneHasher::new();

    let mut data = Vec::new();
    data.extend_from_slice(key);
    data.extend_from_slice(value);
    hasher.leaf_hash(&data)
}

fn verify_intermediate_hashes(
    name: &str,
    db: &CycloneDb<CycloneMode>,
    expected: &super::fixtures::ExpectedHashes,
) {
    // Get twig 0 for single-twig vectors
    // TODO: Access internal twig state for verification

    if let Some(ref hash) = expected.active_bits_mtl3 {
        // Verify active_bits_mtl3
        let expected_hash = parse_hash32(hash);
        // TODO: Compare with actual
    }

    if let Some(ref hash) = expected.left_root {
        let expected_hash = parse_hash32(hash);
        // TODO: Compare with actual
    }

    if let Some(ref hash) = expected.twig_root {
        let expected_hash = parse_hash32(hash);
        // TODO: Compare with actual
    }
}
```

### Step 3: Add Conformance Tests

**File**: `storage/src/qmdb/tests/cyclone_compat_tests.rs` (continued)

```rust
/// Conformance test to detect accidental changes.
#[cfg(feature = "arbitrary")]
mod conformance {
    use crate::qmdb::cyclone_compat::{CycloneDb, CycloneMode};
    use commonware_conformance::conformance_tests;

    // Generate deterministic sequences and hash the results
    struct CycloneDbConformance;

    impl commonware_conformance::Conformance for CycloneDbConformance {
        fn commit(&self, data: &[u8]) -> Vec<u8> {
            let mut db = CycloneDb::<CycloneMode>::new();

            // Use data to generate entries
            for chunk in data.chunks(64) {
                let mut entry_hash = [0u8; 32];
                entry_hash[..chunk.len().min(32)].copy_from_slice(&chunk[..chunk.len().min(32)]);
                db.add(entry_hash);
            }

            db.root().to_vec()
        }
    }

    conformance_tests! {
        CycloneDbConformance => 100,
    }
}
```

### Step 4: Update Test Module

**File**: `storage/src/qmdb/tests/mod.rs`

```rust
mod fixtures;

#[cfg(feature = "cyclone-compat")]
mod cyclone_compat_tests;
```

## Files to Create

- `storage/src/qmdb/tests/fixtures/mod.rs`
- `storage/src/qmdb/tests/cyclone_compat_tests.rs`

## Files to Modify

- `storage/src/qmdb/tests/mod.rs` (add test modules)

## Testing Strategy

### Test Categories

| Category | Count | Description |
|----------|-------|-------------|
| Empty | 1 | Empty database root |
| Single entry | 5+ | Various positions within twig |
| Full twig | 1 | Exactly 2048 entries |
| Multi-twig | 4+ | Cross-twig operations |
| Deactivation | 5+ | Entry deactivation scenarios |
| Edge cases | 5+ | Boundary conditions |

### Verification Levels

1. **State root**: Final root must match exactly
2. **Twig root**: Individual twig roots must match
3. **Active bits tree**: MTL1/2/3 hashes must match
4. **Entry tree**: Left root must match

### CI Integration

```yaml
# In .github/workflows/test.yml
- name: Run cyclone compatibility tests
  run: cargo test --features cyclone-compat cyclone_compat
```

## Dependencies

- Task 1 (Test vectors must exist)
- Task 5 (CycloneDb implementation)

## Estimated Effort

- 1 day for test infrastructure
- 0.5 day for test implementation
- 0.5 day for CI integration
