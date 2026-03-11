# Trust Chain

The central challenge of off-chain verification is: how does a verifier trust
that a Merkle root is authentic? The answer is a chain of Merkle proofs, two
levels deep, anchored to an institutionally provided UTxO set root.

## The chain

```
UTxO Set Merkle Root (cardano-utxo-csmt, published by Cardano institution)
  └─ Cage UTxO exists (CSMT inclusion proof)
       └─ Credential Trie Root (in cage UTxO datum)
            └─ Credential exists (MPF inclusion proof)
```

Each level is independently verifiable. A verifier traverses the chain from top
to bottom, checking each proof against the level above.

## Level 1: UTxO set Merkle tree (CSMT)

[cardano-utxo-csmt][csmt] maintains a Compact Sparse Merkle Tree over
Cardano's entire UTxO set. It synchronizes with a Cardano node in real time,
tracking every UTxO creation and consumption.

The CSMT root is published by a trusted institution (e.g. the Cardano
Foundation). Verifiers trust this root based on the institution's reputation and
operational integrity — the same kind of institutional trust that underpins
certificate authorities in TLS.

Given a trusted CSMT root, an inclusion proof demonstrates that a specific UTxO
exists in the set:

- The cage UTxO (identified by transaction ID + output index) **exists**
- The UTxO is at the expected script address
- The UTxO datum contains the expected credential trie root

**Proof size**: O(log n) where n is the total number of UTxOs on Cardano
(currently ~15 million).

### Root distribution

The institution publishes the current CSMT root via an HTTP API. Verifiers
query this endpoint to obtain the root they verify proofs against. The root
changes with every block as UTxOs are created and consumed.

Multiple institutions could independently run cardano-utxo-csmt instances and
publish roots. A verifier can cross-check roots from multiple sources for
additional assurance — if two independent instances agree on the root, the
probability of corruption is negligible.

## Level 2: Credential trie (MPF)

The cage UTxO datum contains the credential trie root (32 bytes, Blake2b-256).
An MPF (Merkle Patricia Forestry) inclusion proof demonstrates that a specific
credential exists in the issuer's trie.

The proof demonstrates:

- The credential key exists in the trie
- The credential value matches the expected data
- The proof verifies against the root from the cage UTxO datum

**Proof size**: O(log n) where n is the number of credentials in the issuer's
trie.

## Complete verification

A verifier with a proof bundle performs:

```
verify(csmtRoot, csmtProof, cageUtxo, credentialProof, key, value):

  1. Obtain csmtRoot from trusted institution
     → the current UTxO set Merkle root

  2. Verify csmtProof for cageUtxo against csmtRoot
     → confirm cage UTxO exists, extract datum

  3. Extract credentialRoot from datum

  4. Verify credentialProof for (key, value) against credentialRoot
     → confirm credential exists with expected value

  5. Decode value, check expiration, schema, trust policy
     → accept or reject
```

Total computation: two rounds of hash verification plus one HTTP request for the
root. Runnable on any device.

## Proof sizes

| Level | Tree size | Proof steps | Hash size |
|-------|-----------|-------------|-----------|
| CSMT | ~15M UTxOs | ~24 | 32 bytes |
| MPF | Varies per issuer | ~20 (for 1M credentials) | 32 bytes |

Total proof bundle: approximately 1-2 KB for a single credential verification
(excluding the CSMT root itself, which is obtained separately).

## Non-membership proofs

The same trust chain supports non-membership proofs at every level:

- **CSMT**: prove a UTxO does NOT exist (cage has been destroyed)
- **MPF**: prove a credential does NOT exist (revoked or never issued)

This enables verifiers to prove negative claims — a capability that EAS on
Ethereum fundamentally cannot offer.

## Freshness

The CSMT root reflects a specific point in time (a specific slot/block).
Credentials issued or revoked after the root was published will not be
reflected until the root is updated.

Verifiers control the tradeoff:

- **High freshness**: query the institution's API for the latest root before
  each verification
- **Cached**: use a recently fetched root, tolerating some staleness
- **Mixed**: use cached roots for existence proofs (credential was valid at
  slot X), query fresh roots for revocation checks (is it still valid now?)

The cardano-utxo-csmt service updates its root with every new block (~20
seconds on Cardano mainnet), so freshness is bounded by block time.

[csmt]: https://github.com/cardano-foundation/cardano-utxo-csmt
