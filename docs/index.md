# Cardano VCR

**Verifiable Credential Registry** — a [W3C Verifiable Credentials Data Model 2.0][w3c-vc]
implementation on Cardano, built on [Merkle Patricia Forestry (MPFS)][mpfs].

## What is this?

Cardano VCR provides a protocol for issuing, storing, revoking, and
cryptographically verifying credentials on the Cardano blockchain. It implements
the W3C VC standard using Merkle Patricia Tries as the underlying data structure,
enabling compact on-chain storage and efficient proof generation.

## Key properties

- **W3C compliant** — implements the Verifiable Credentials Data Model 2.0, not a
  proprietary attestation format
- **Merkle proof verification** — any Plutus validator or off-chain entity can
  verify credentials via compact inclusion proofs
- **Proof of non-membership** — can prove a credential does NOT exist (revocation
  verification, non-attestation proof)
- **Multi-authority schemas** — independent schema authorities compete on trust;
  no single entity controls the schema layer
- **Off-chain verification** — credentials verifiable without a Cardano node,
  via Merkle proof chains anchored to institutionally published UTxO set roots

## How it works

Cardano VCR uses three tiers of [MPFS cages](architecture/three-tier-cages.md):

1. **Schema authorities** manage schema registries (mostly append-only)
2. **Credential issuers** manage credential tries (insert + revoke)
3. **Verifiers** consume Merkle proofs (on-chain or off-chain)

Each tier is a separate MPFS cage instance. Cages are independent — a compromised
or rogue authority only affects its own data. Trust is a policy decision made by
verifiers, not enforced by the protocol.

## Components

| Component | Repository | Status |
|-----------|-----------|--------|
| On-chain validators (Aiken) | [cardano-mpfs-onchain][onchain] | Working |
| Off-chain service (Haskell) | [cardano-mpfs-offchain][offchain] | Working |
| Merkle trie library | [haskell-mts][mts] | Working |
| UTxO set Merkle tree | [cardano-utxo-csmt][csmt] | Working |
| VCR protocol layer | This repository | Design phase |

[w3c-vc]: https://www.w3.org/TR/vc-data-model-2.0/
[mpfs]: https://github.com/cardano-foundation/cardano-mpfs-onchain
[onchain]: https://github.com/cardano-foundation/cardano-mpfs-onchain
[offchain]: https://github.com/paolino/cardano-mpfs-offchain
[mts]: https://github.com/paolino/haskell-mts
[csmt]: https://github.com/cardano-foundation/cardano-utxo-csmt
