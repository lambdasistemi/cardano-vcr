# Comparison with Supply Chain Security Systems

Beyond [EAS](eas-comparison.md) in the blockchain space, there is a mature
ecosystem of non-blockchain systems that use Merkle trees and cryptographic
attestations for trust — specifically in the **software supply chain security**
domain. These systems validate the same core idea behind Cardano VCR: that
Merkle-tree-based attestation registries are a proven pattern for verifiable
claims.

This page compares Cardano VCR with four key systems:
[Sigstore/Rekor](#sigstore-and-rekor), [TUF](#tuf-the-update-framework), and
[in-toto](#in-toto).

## Sigstore and Rekor

[Sigstore][sigstore] is the dominant open-source solution for software artifact
signing. It consists of three components:

| Component | Purpose |
|-----------|---------|
| **Cosign** | Client tool for signing and verifying artifacts |
| **Fulcio** | Certificate authority — issues short-lived signing certificates tied to OIDC identities (GitHub Actions, Google, etc.) |
| **Rekor** | Transparency log — append-only Merkle tree of signed attestations |

### How Sigstore works

1. A CI system (e.g. GitHub Actions) authenticates via OIDC
2. **Fulcio** issues a short-lived certificate binding the CI identity to an
   ephemeral signing key
3. The artifact is signed with the ephemeral key
4. The signature + certificate are recorded in **Rekor** (append-only
   transparency log)
5. A verifier checks: (a) the signature is valid, (b) the certificate was
   issued by Fulcio, (c) the entry exists in Rekor's Merkle tree

The private key only exists in memory during signing — it is never stored. Trust
is anchored to the OIDC identity provider and the Rekor transparency log.

### Rekor: the closest analog to MPFS cages

Rekor is structurally very similar to an MPFS cage in append-only mode:

| Property | Rekor | MPFS Cage (append-only) |
|----------|-------|------------------------|
| Data structure | Merkle tree (append-only log) | Merkle Patricia Trie (append-only mode) |
| Entries | Signed attestations (signature + certificate + artifact hash) | Credential values (claims + metadata) |
| Inclusion proofs | Yes — prove an entry exists in the log | Yes — prove a credential exists in the trie |
| Non-membership proofs | No (log structure doesn't support this) | **Yes** — prove a credential does NOT exist |
| Append-only guarantee | Server-side policy + auditor monitoring | **On-chain validator enforcement** |
| Trust anchor | Rekor server operator (currently Sigstore project) | Cage oracle identity (on-chain, verifiable) |
| Tamper evidence | Merkle tree consistency proofs + signed tree heads | Blockchain immutability + validator rules |
| Deletion | Not possible (log is append-only) | Configurable per cage (can be forbidden) |

### Key differences

**Trust model**: Rekor requires trusting the Rekor server operator. If the
operator is compromised, they could present different views of the log to
different users (a "split-view" attack). Auditors can detect this, but only
after the fact. In Cardano VCR, the cage root is on-chain — there is one
canonical state visible to everyone. Split-view attacks are impossible because
the blockchain provides a single source of truth.

**Append-only enforcement**: Rekor's append-only property is a server policy,
enforced by auditor monitoring. An MPFS cage's append-only property (when
configured) is enforced by the **Plutus validator** — the blockchain rejects
transactions that violate it. This is a stronger guarantee: enforcement is
cryptographic and automated, not dependent on monitoring.

**Non-membership proofs**: Rekor cannot prove that an entry does NOT exist in
the log (the log structure only supports inclusion proofs). MPFS tries support
non-membership proofs natively. This matters for revocation — you can prove a
credential was revoked, not just that it once existed.

**Decentralization**: Rekor is operated by a single entity (the Sigstore
project). MPFS cages are operated by independent entities, each running their
own instance. There is no central log.

## TUF (The Update Framework)

[TUF][tuf] is a framework for securing software update systems. It defines a
role-based trust model with key delegation and threshold signatures.

### TUF architecture

TUF defines four top-level roles:

| Role | Purpose |
|------|---------|
| **Root** | Trusted keys for all other roles. Rarely updated, highest security. |
| **Targets** | Signs the actual software artifacts. Can delegate to sub-roles. |
| **Snapshot** | Signs a consistent view of all target metadata. |
| **Timestamp** | Signs the current snapshot, providing freshness guarantees. |

Key properties:

- **Key delegation**: the Targets role can delegate trust to sub-roles, each
  responsible for a subset of artifacts. Delegation can be further nested.
- **Threshold signatures**: each role requires M-of-N signatures, so
  compromising a single key is insufficient.
- **Compromise resilience**: separating roles means a compromised Timestamp key
  (rotated frequently) doesn't compromise the Root key (stored offline).

### Mapping to Cardano VCR

TUF's role hierarchy maps closely to VCR's three-tier cage architecture:

| TUF Concept | Cardano VCR Equivalent |
|-------------|----------------------|
| Root role | Schema authority cage (trusted root of the hierarchy) |
| Targets role | Credential issuer cage (signs/issues the actual credentials) |
| Key delegation | Schema authority registers schemas; issuers reference them |
| Threshold signatures | Multi-signature resolvers (require M-of-N signatories) |
| Snapshot/Timestamp | Cage root on-chain (provides both consistency and freshness) |
| Compromise resilience | Independent cages — compromising one doesn't affect others |

### Key differences

**Scope**: TUF secures software update distribution. Cardano VCR secures
arbitrary verifiable credentials. TUF's domain is narrower but its role model
is more elaborate (four mandatory roles with specific rotation policies).

**Key management**: TUF's strength is its detailed key rotation and delegation
model. Cardano VCR delegates this to the cage oracle's key management. TUF is
more prescriptive about how keys should be managed; VCR leaves this to the
issuer.

**Freshness**: TUF uses a dedicated Timestamp role to guarantee clients see the
latest metadata. In Cardano VCR, freshness comes from the blockchain itself —
the cage UTxO's datum is the latest state, and the block timestamp provides
temporal ordering.

**Offline verification**: Both support offline verification. TUF clients cache
metadata and verify signatures locally. VCR verifiers cache CSMT roots and
verify Merkle proofs locally.

## in-toto

[in-toto][intoto] is a framework for securing the integrity of software supply
chains. It focuses on **process attestation** — proving that specific steps were
performed by authorized parties in the correct order.

### in-toto architecture

| Concept | Purpose |
|---------|---------|
| **Layout** | A project owner defines the supply chain: steps, authorized functionaries, and artifact flow rules |
| **Link metadata** | Each functionary records what they did: command, input artifacts, output artifacts |
| **Attestation** | A signed statement about a supply chain step (e.g. "source was reviewed", "build was reproducible") |

The layout chains links together: step A's outputs must match step B's inputs.
A verifier checks that (a) each step was performed by an authorized functionary,
(b) artifacts flow correctly between steps, and (c) no unauthorized
modifications occurred.

### Mapping to Cardano VCR

in-toto's attestation model maps to VCR's referenced credentials:

| in-toto Concept | Cardano VCR Equivalent |
|-----------------|----------------------|
| Layout (supply chain definition) | Schema (defines what a credential must contain) |
| Functionary | Credential issuer (authorized to perform a step) |
| Link metadata | Credential (attests that a step was performed) |
| Artifact chaining (step A output → step B input) | Referenced credentials (`refUID` — credential B references credential A) |
| Project owner | Schema authority (defines the process) |
| Verification | Merkle proof chain across multiple issuer cages |

### Key differences

**Process vs state**: in-toto attests to a **process** (a sequence of steps).
Cardano VCR attests to **state** (a set of claims at a point in time). in-toto's
chaining model (step outputs → step inputs) has no direct VCR equivalent,
though `refUID` provides a weaker form of linkage.

**Artifact binding**: in-toto binds attestations to specific file hashes. VCR
credentials are more general — they can attest to anything, not just software
artifacts.

**Enforcement**: in-toto's layout enforcement is done at verification time by
the client. The layout is a policy document, not an on-chain constraint. In VCR,
schema and resolver enforcement happens at issuance time via on-chain validators.

## Cross-cutting comparison

| Property | Rekor | TUF | in-toto | Cardano VCR |
|----------|-------|-----|---------|-------------|
| **Data structure** | Append-only Merkle log | Signed metadata files | Signed link metadata | Merkle Patricia Trie |
| **Inclusion proofs** | Yes | No (metadata is downloaded whole) | No | Yes |
| **Non-membership proofs** | No | No | No | **Yes** |
| **Trust anchor** | Rekor server operator | Root keys (offline) | Project owner keys | On-chain cage UTxO |
| **Append-only enforcement** | Server policy + auditors | N/A | N/A | On-chain validator |
| **Key delegation** | Via Fulcio (OIDC) | Explicit role hierarchy | Layout defines functionaries | Schema authorities + resolvers |
| **Decentralized** | No (single operator) | No (single repo) | No (single project owner) | **Yes** (independent cages) |
| **Split-view resistance** | Auditor-dependent | N/A | N/A | **Blockchain consensus** |
| **Revocation** | Not supported | Key rotation | N/A | Delete from trie |
| **Offline verification** | Inclusion proof + cached tree head | Cached metadata | Cached layout + links | Cached CSMT root + Merkle proofs |
| **Standard** | Custom (OpenSSF) | Custom (CNCF/Linux Foundation) | [SLSA][slsa] provenance format | [W3C VC Data Model 2.0][w3c-vc] |

## What Cardano VCR learns from these systems

### From Rekor

The transparency log model validates the core VCR design: a Merkle tree of
attestations with inclusion proofs, operated by an authority, verifiable by
anyone. VCR adds non-membership proofs, decentralization (multiple independent
cages), and on-chain enforcement of append-only policies.

### From TUF

The role-based key delegation model is more sophisticated than VCR's current
design. VCR could benefit from formalizing key rotation policies for schema
authorities and credential issuers, and from supporting threshold signatures
natively (currently available via multi-signature resolvers).

### From in-toto

The process attestation model (chaining steps via artifact hashes) suggests
that VCR's `refUID` mechanism could be formalized into a **credential chain**
pattern — a sequence of credentials where each one references the previous,
forming a verifiable supply chain or workflow audit trail.

[sigstore]: https://docs.sigstore.dev/
[rekor]: https://docs.sigstore.dev/logging/overview/
[tuf]: https://theupdateframework.io/
[intoto]: https://in-toto.io/
[slsa]: https://slsa.dev/
[w3c-vc]: https://www.w3.org/TR/vc-data-model-2.0/
