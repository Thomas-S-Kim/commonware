# Task 6: Compatibility Tests

## Current State

- Test vectors from Task 1 exist
- No tests consuming them
- No verification of state root equivalence

## Goal

Test suite validating:
1. `BalancedTreeScheme` against reference test vectors
2. `MmrScheme` against commonware test vectors
3. Scheme selection via generics
4. Default scheme resolution

## Execution

### Step 1: Vector Loader

**File**: `storage/src/qmdb/tests/fixtures/mod.rs`

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum FixtureError {
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
    #[error("TOML parse error: {0}")]
    Toml(#[from] toml::de::Error),
    #[error("hex decode error: {0}")]
    Hex(#[from] hex::FromHexError),
    #[error("invalid hash length: expected 32, got {0}")]
    InvalidHashLength(usize),
}

#[derive(Debug, Deserialize)]
pub struct TestVector {
    pub metadata: VectorMetadata,
    pub entries: Vec<VectorEntry>,
    pub expected_hashes: ExpectedHashes,
}

#[derive(Debug, Deserialize)]
pub struct VectorEntry {
    pub key: String,
    pub value: String,
    pub serial_number: u64,
    #[serde(default)]
    pub deactivated: bool,
}

#[derive(Debug, Deserialize)]
pub struct ExpectedHashes {
    pub state_root: String,
    pub segment_root: Option<String>,
    pub entry_root: Option<String>,
    pub hierarchical_bitmap_l3: Option<String>,
}

impl TestVector {
    pub fn load(path: &str) -> Result<Self, FixtureError> {
        let content = std::fs::read_to_string(path)?;
        Ok(toml::from_str(&content)?)
    }

    pub fn load_all() -> Result<Vec<(String, Self)>, FixtureError> {
        let dir = concat!(env!("CARGO_MANIFEST_DIR"), "/src/qmdb/tests/fixtures/balanced_tree");
        let mut results = Vec::new();
        for entry in std::fs::read_dir(dir)? {
            let entry = entry?;
            let path = entry.path();
            if path.extension().map_or(false, |ext| ext == "toml") {
                let name = path
                    .file_stem()
                    .and_then(|s| s.to_str())
                    .map(|s| s.to_string())
                    .unwrap_or_default();
                let vector = Self::load(path.to_str().unwrap_or_default())?;
                results.push((name, vector));
            }
        }
        Ok(results)
    }
}

pub fn parse_hash32(s: &str) -> Result<[u8; 32], FixtureError> {
    let s = s.strip_prefix("0x").unwrap_or(s);
    let bytes = hex::decode(s)?;
    bytes
        .try_into()
        .map_err(|v: Vec<u8>| FixtureError::InvalidHashLength(v.len()))
}
```

### Step 2: Compatibility Tests

**File**: `storage/src/qmdb/tests/balanced_tree_tests.rs`

```rust
use super::fixtures::{parse_hash32, FixtureError, TestVector};

#[test]
fn test_all_vectors() {
    let vectors = TestVector::load_all().expect("failed to load test vectors");
    assert!(!vectors.is_empty(), "no test vectors found");

    for (name, vector) in vectors {
        test_vector(&name, &vector).expect(&format!("test vector {} failed", name));
    }
}

fn test_vector(name: &str, vector: &TestVector) -> Result<(), FixtureError> {
    let mut db = Qmdb::<BalancedTreeScheme>::new();

    for entry in &vector.entries {
        let key = parse_hash32(&entry.key)?;
        let value = parse_hash32(&entry.value)?;
        let hash = compute_entry_hash(&key, &value);
        let sn = db.add(hash.into()).expect("add failed");
        assert_eq!(sn, entry.serial_number, "serial number mismatch in {}", name);

        if entry.deactivated {
            db.set_inactive(sn).expect("set_inactive failed");
        }
    }

    let root = db.root();
    let expected = parse_hash32(&vector.expected_hashes.state_root)?;
    assert_eq!(root.0, expected, "root mismatch in {}", name);
    Ok(())
}

/// Test proof generation and verification with unified API
#[test]
fn test_proof_roundtrip() {
    let mut db = Qmdb::<BalancedTreeScheme>::new();

    // Add some entries
    let entry1 = [1u8; 32].into();
    let entry2 = [2u8; 32].into();
    let entry3 = [3u8; 32].into();

    let sn1 = db.add(entry1).unwrap();
    let sn2 = db.add(entry2).unwrap();
    let sn3 = db.add(entry3).unwrap();

    // Get root
    let root = db.root();

    // Generate and verify proofs (unified API)
    let proof1 = db.prove(sn1).unwrap();
    let proof2 = db.prove(sn2).unwrap();
    let proof3 = db.prove(sn3).unwrap();

    assert!(proof1.verify(&entry1, &root).unwrap());
    assert!(proof2.verify(&entry2, &root).unwrap());
    assert!(proof3.verify(&entry3, &root).unwrap());

    // All should be active
    assert!(proof1.is_active());
    assert!(proof2.is_active());
    assert!(proof3.is_active());

    // Deactivate one
    db.set_inactive(sn2).unwrap();
    let new_root = db.root();

    // Old proofs should fail against new root
    assert!(!proof2.verify(&entry2, &new_root).unwrap());

    // New proof shows inactive
    let new_proof2 = db.prove(sn2).unwrap();
    assert!(!new_proof2.is_active());
    assert!(new_proof2.verify(&entry2, &new_root).unwrap());
}

/// Test that the same code works for both schemes (swap via type alias)
fn generic_scheme_test<S: StateRootScheme>()
where
    S::Digest: From<[u8; 32]>,
{
    let mut db: Qmdb<S> = Qmdb::new();

    let entry: S::Digest = [42u8; 32].into();
    let sn = db.add(entry.clone()).unwrap();

    let root = db.root();
    let proof = db.prove(sn).unwrap();

    // Unified API works for any scheme
    assert!(proof.verify(&entry, &root).unwrap());
    assert!(proof.is_active());
}

#[test]
fn test_balanced_tree_generic() {
    generic_scheme_test::<BalancedTreeScheme>();
}

#[test]
#[cfg(feature = "mmr")]
fn test_mmr_generic() {
    generic_scheme_test::<MmrScheme<Sha256, 1024>>();
}
```

### Step 3: Module Setup

**File**: `storage/src/qmdb/tests/mod.rs`

```rust
mod fixtures;

#[cfg(feature = "balanced_tree")]
mod balanced_tree_tests;
```

## Files to Create

- `storage/src/qmdb/tests/fixtures/mod.rs`
- `storage/src/qmdb/tests/balanced_tree_tests.rs`

## Files to Modify

- `storage/src/qmdb/tests/mod.rs`

## Testing

```bash
cargo test -p commonware-storage --features balanced_tree balanced_tree
```

## Dependencies

- Task 1 (test vectors)
- Task 5 (StateRootScheme)
