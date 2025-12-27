1. The commonware-storage QMDB root is equivalent to the original QMDB’s shard root. The original QMDB’s global root is the concatenation of all shard roots plus the compact parameters.

Yes. So the best we can do is to make state root calculation and proof verification/generation compatible in one shard.

2. The compact parameters are required for sub-block / block-range proof recursion, which is not possible with commonware-storage QMDB’s “compact per commit” design.

Compaction scheme can be different. We can just make state root calculation and proof verification/generation compatible.

3. Parameters are compile-time constants (not runtime-configurable). Changing grafting height, twig size, or hash function is a breaking change for both.

But in commonware these are generic constants which are configurable.

4. Commonware-storage QMDB instead appends a partial_trunk after grafting the MMR.

This is the first trouble.

Today, commonware computes mmr_root using the following root function:

```
pub trait Hasher<D: Digest>: Send + Sync {
    type Inner: commonware_cryptography::Hasher<Digest = D>;
    fn leaf_digest(&mut self, pos: Position, element: &[u8]) -> D;
    fn node_digest(&mut self, pos: Position, left: &D, right: &D) -> D;
    fn root<'a>(&mut self, size: Position, peak_digests: impl Iterator<Item = &'a D>) -> D;
    fn digest(&mut self, data: &[u8]) -> D;
    fn inner(&mut self) -> &mut Self::Inner;
    fn fork(&self) -> impl Hasher<D>;
}
```

Then it calculates state root as: `H(mmr_root || next_bit || last_chunk_digest)`

If we change the root function like following:
```
fn root<'a>(&mut self, size: Position, next_bit, last_chunk_digest, peak_digests: impl Iterator<Item = &'a D>) -> D;
```

I think it can generate the same state root as our qmdb.

5. When N=2048 (same as original QMDB), it is `H(2048 bits | 256 bits)` / `H(bits | entries tree hash)`. For original QMDB: `H(256 bits | 256 bits)` / `H(entries tree hash | bits tree hash)`

This is the second trouble. We need to add the following function to Hasher:

```
fn graft_digest(&mut self, pos: Position, peak: &D, chunk_bits) -> D;
```

In this function, we can builds a 3-level binary tree to compute active_bits_root_hash.