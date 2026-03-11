# Trust Chain

The central challenge of off-chain verification is: how does a verifier trust
that a Merkle root is authentic? The answer is a chain of Merkle proofs, three
levels deep, anchored to Cardano's consensus.

## The chain

```
Mithril Certificate (SPO quorum signature)
  └─ UTxO Set Merkle Root (cardano-utxo-csmt)
       └─ Cage UTxO exists (CSMT inclusion proof)
            └─ Credential Trie Root (in cage UTxO datum)
                 └─ Credential exists (MPF inclusion proof)
```

Each level is independently verifiable. A verifier traverses the chain from top
to bottom, checking each proof against the level above.

## Level 1: Mithril certificate

[Mithril][mithril] is a stake-based threshold multi-signature protocol for
Cardano. Stake pool operators (SPOs) sign snapshots of the ledger state. A
certificate is valid when a quorum of stake-weighted signatures is reached.

The Mithril certificate provides:

- A **UTxO set Merkle root**: the root of a Merkle tree containing every UTxO
  in the Cardano ledger at a specific slot
- **Slot number**: when the snapshot was taken
- **SPO signatures**: cryptographic proof of consensus

**Trust basis**: the same set of SPOs that produce Cardano blocks. If you trust
Cardano's consensus, you trust Mithril certificates.

## Level 2: UTxO set Merkle tree (CSMT)

[cardano-utxo-csmt][csmt] maintains a Compact Sparse Merkle Tree over
Cardano's entire UTxO set. Given the Mithril-certified root, a CSMT inclusion
proof demonstrates that a specific UTxO exists in the set.

The proof demonstrates:

- The cage UTxO (identified by transaction ID + output index) **exists**
- The UTxO is at the expected script address
- The UTxO datum contains the expected credential trie root

**Proof size**: O(log n) where n is the total number of UTxOs on Cardano
(currently ~15 million).

!!! warning "Current status"
    The bridge between Mithril certificates and cardano-utxo-csmt is **not yet
    implemented**. This is the critical gap in the trust chain. The CSMT service
    works independently (maintains the tree, generates proofs), but does not yet
    consume Mithril-certified roots.

## Level 3: Credential trie (MPF)

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
verify(mithrilCert, csmtProof, cageUtxo, credentialProof, key, value):

  1. Check mithrilCert.signatures against known SPO keys
     → obtain utxoSetRoot

  2. Verify csmtProof for cageUtxo against utxoSetRoot
     → confirm cage UTxO exists, extract datum

  3. Extract credentialRoot from datum

  4. Verify credentialProof for (key, value) against credentialRoot
     → confirm credential exists with expected value

  5. Decode value, check expiration, schema, trust policy
     → accept or reject
```

Total computation: three rounds of hash verification. No network access. No
Cardano node. Runnable on any device.

## Proof sizes

| Level | Tree size | Proof steps | Hash size |
|-------|-----------|-------------|-----------|
| Mithril | ~3,000 SPOs | ~12 | 32 bytes |
| CSMT | ~15M UTxOs | ~24 | 32 bytes |
| MPF | Varies per issuer | ~20 (for 1M credentials) | 32 bytes |

Total proof bundle: approximately 2-3 KB for a single credential verification.

## Non-membership proofs

The same trust chain supports non-membership proofs at every level:

- **CSMT**: prove a UTxO does NOT exist (cage has been destroyed)
- **MPF**: prove a credential does NOT exist (revoked or never issued)

This enables verifiers to prove negative claims — a capability that EAS on
Ethereum fundamentally cannot offer.

## Freshness vs completeness

The trust chain provides a snapshot at a specific slot. Credentials issued or
revoked after the Mithril snapshot will not be reflected.

Verifiers control the tradeoff:

- **High freshness**: require recent Mithril certificates (< 1 hour old),
  or fall back to direct chain query
- **High availability**: accept older certificates, tolerating some staleness
- **Mixed**: use Mithril for existence proofs (credential was valid at slot X),
  use direct query for revocation checks (is it still valid now?)

[mithril]: https://github.com/input-output-hk/mithril
[csmt]: https://github.com/cardano-foundation/cardano-utxo-csmt
