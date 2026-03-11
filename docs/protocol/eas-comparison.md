# Comparison with EAS

The [Ethereum Attestation Service (EAS)][eas] is the most widely used attestation
system on Ethereum. This page compares EAS with Cardano VCR to clarify the
design differences and tradeoffs.

!!! note
    Cardano VCR is not an EAS clone. Both address the same problem space
    (verifiable claims) but take fundamentally different approaches. Cardano VCR
    implements the W3C Verifiable Credentials Data Model 2.0; EAS is an
    Ethereum-specific system that predates and does not fully follow the W3C
    standard.

## EAS architecture

EAS consists of two Solidity smart contracts:

1. **SchemaRegistry** — registers attestation schemas (data templates). Schemas
   are immutable once registered. A schema cannot be created if one with the
   same structure already exists.

2. **EAS contract** — creates attestations against registered schemas.

### EAS attestation fields

| Field | Description |
|-------|-------------|
| `recipient` | Ethereum address receiving the attestation |
| `attester` | Address creating the attestation |
| `expirationTime` | Unix timestamp (0 = no expiration) |
| `revocable` | Boolean — can this attestation be revoked? |
| `refUID` | Reference to another attestation's UID |
| `data` | Encoded data matching the schema |
| `schema` | Schema UID |
| `value` | ETH to send to resolver |

### EAS features

- **Permissionless** — anyone can call `attest()` with no intermediary
- **On-chain and off-chain** attestations
- **Resolver contracts** — optional smart contracts for custom logic
- **Referenced attestations** — attestations can point to other attestations
- **Delegated attestations** — attest on behalf of another address

## Fundamental difference: writing vs reading

EAS optimizes for **writing** — making it easy for anyone to create attestations
permissionlessly. Cardano VCR optimizes for **reading** — making it easy for
anyone to verify credentials efficiently.

This reflects a deeper question: what is the point of an attestation if no
smart contract can efficiently consume it?

### The composability problem in EAS

In EAS, attestations are stored as entries in a Solidity mapping. To use an
attestation from another smart contract, you need a **cross-contract call**:

```solidity
// Contract B needs to check an attestation
Attestation memory att = eas.getAttestation(uid);
require(att.attester == trustedIssuer);
```

This has significant limitations:

- **Gas-expensive** — SLOAD + cross-contract call overhead
- **Same-chain only** — cannot verify across chains or L2s
- **Off-chain attestations are invisible** — they cannot be referenced from
  smart contracts at all
- **No proof of non-existence** — you cannot prove that an attestation does NOT
  exist

### Composability in Cardano VCR

In Cardano VCR, any Plutus validator can verify a credential in the same
transaction:

1. Include the issuer's cage UTxO as a **reference input** (provides the
   credential root)
2. Include a **Merkle proof** in the transaction redeemer
3. The validator verifies the proof — O(log n) hash computations

This is:

- **Cheap** — hash computations, no cross-contract calls
- **Cross-cage** — verify credentials from multiple issuers in one transaction
- **Off-chain compatible** — the same proof works off-chain
- **Non-membership provable** — can prove a credential was revoked or never
  existed

## Feature comparison

| Feature | EAS | Cardano VCR |
|---------|-----|-------------|
| Self-attestation | Permissionless, one tx | Requires oracle (issuer) |
| Cross-contract verification | SLOAD + call (expensive) | Reference input + Merkle proof (cheap) |
| Off-chain attestation usable on-chain | No | Yes (proof against anchored root) |
| Proof of non-existence | Impossible | Yes (non-membership proof) |
| On-chain storage per credential | Full attestation record | Single 32-byte root for all credentials |
| Batch operations | One tx per attestation | Multiple operations folded into one proof |
| Revocation model | Flag (data preserved forever) | Deletion from trie (true removal) |
| Schema governance | Single global contract | Multiple independent authorities |
| Expiration cleanup | Expired attestations stay on-chain | Can be deleted from trie |
| Off-chain verification (no node) | Not supported | Supported via Merkle proof chain |
| W3C VC compliance | Partial | Full (Data Model 2.0) |

## The oracle tradeoff

The most visible difference is that EAS allows permissionless self-attestation
while Cardano VCR requires an oracle (issuer) to process credential operations.

This is a consequence of the eUTxO model: someone must hold the full trie state
to compute Merkle proofs, and someone must consume the cage UTxO to update the
root. On Ethereum's account model, the contract manages its own state and
anyone can write to it.

However, this is not a limitation in practice:

1. **The W3C VC model assumes authoritative issuers.** A university issues
   degrees. A government issues IDs. The oracle model matches the real-world
   trust model.

2. **Permissionless self-attestation has limited value.** "I attest that I am
   trustworthy" carries no weight without reputation behind it. The use cases
   where attestations matter most are inherently oracle-driven.

3. **The MPFS 3-phase protocol provides delegation.** A requester submits a
   request, the oracle processes it. If the oracle fails to act, the requester
   can retract. This is more robust than EAS delegation, which has no timeout
   guarantees.

## Standards alignment

EAS is an Ethereum-specific system. It does not implement the W3C VC standard,
though there is an [open issue requesting W3C VC support][eas-w3c-issue].

Cardano VCR implements the W3C Verifiable Credentials Data Model 2.0 directly.
Every concept in the specification (issuer, holder, verifier, verifiable data
registry, credential schema, credential status, verifiable presentation) has a
concrete mapping to the protocol.

See [W3C Alignment](w3c-alignment.md) for the complete mapping.

[eas]: https://attest.org/
[eas-w3c-issue]: https://github.com/ethereum-attestation-service/eas-contracts/issues/4
