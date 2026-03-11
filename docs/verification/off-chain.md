# Off-chain Verification

Off-chain verification allows any entity — a phone, a browser, an IoT device —
to verify a credential without running a Cardano node or submitting a
transaction. The verifier only needs the ability to compute Blake2b-256 hashes.

## Proof bundle

A holder presents a **proof bundle** to a verifier:

```
ProofBundle:
  credential:
    key   : ByteArray           -- Credential ID
    value : ByteArray           -- Credential data (or hash, for hybrid mode)
  issuer:
    cageToken : TokenId         -- Identifies the issuer
    root      : ByteArray       -- The issuer's current trie root (32 bytes)
  proof   : [ProofStep]         -- Merkle proof path
  schema  : Maybe SchemaProof   -- Optional schema verification
```

The verifier performs:

1. **Proof verification**: verify the Merkle proof for `(key, value)` against
   `root`. This is pure computation — hash the proof steps and check the result
   equals the root.

2. **Credential inspection**: decode the value, check expiration, schema
   reference, and any application-specific fields.

3. **Trust check**: does the verifier trust this `cageToken` as a credential
   issuer? This is a policy decision.

4. **Root trust**: does the verifier trust that `root` is the authentic current
   root for this cage? This is the critical question — see
   [Trust Chain](trust-chain.md).

## Root distribution

The proof bundle includes a root, but the verifier must trust that root.
Several trust levels are possible, from strongest to weakest:

### Level 1: Direct chain query

The verifier queries a Cardano node (via N2C or an API) for the cage UTxO and
reads the root from its datum. This is authoritative but requires network access
and a trusted node.

### Level 2: Mithril-certified root

The verifier obtains a Mithril certificate (signed by a quorum of stake pool
operators) that certifies the UTxO set. It then verifies the cage UTxO's
existence in the certified set and reads the root. No full node required — only
Mithril certificate verification.

See [Trust Chain](trust-chain.md) for the full Mithril verification path.

### Level 3: Issuer-signed root

The issuer periodically signs their current root and publishes it (on a website,
via an API, etc.). The verifier trusts the issuer's signature. This is the
simplest approach but relies on the issuer's key management.

### Level 4: Holder-provided root (weakest)

The verifier trusts the root provided in the proof bundle. This is only
appropriate when the verifier has other means of establishing trust (e.g. the
holder is presenting in person with government-issued ID).

## Verification without a node

The complete off-chain verification flow, assuming Level 2 (Mithril):

```mermaid
sequenceDiagram
    participant H as Holder
    participant V as Verifier
    participant M as Mithril Certificate

    H->>V: Present proof bundle
    V->>M: Verify Mithril certificate (SPO quorum signatures)
    V->>V: Verify cage UTxO exists in certified UTxO set (CSMT proof)
    V->>V: Extract root from cage UTxO datum
    V->>V: Verify credential Merkle proof against root
    V->>V: Check expiration, schema, trust policy
    V->>V: Accept or reject
```

At no point does the verifier contact a Cardano node. The entire verification
is local computation over cryptographic proofs.

## Offline verification

If the verifier has a cached Mithril certificate and CSMT snapshot, verification
is fully offline. The tradeoff is freshness — the cached root may be stale. A
credential revoked after the snapshot was taken will still appear valid.

For time-sensitive credentials, the verifier can set a maximum age for the
Mithril certificate (e.g. "I only accept certificates less than 1 hour old").

## Multiple credentials

Off-chain verifiable presentations bundle multiple proof bundles:

```
Presentation:
  credentials:
    - { issuer: A, key: ..., value: ..., proof: [...] }
    - { issuer: B, key: ..., value: ..., proof: [...] }
  roots:
    - { cageToken: A, root: ..., csmtProof: [...] }
    - { cageToken: B, root: ..., csmtProof: [...] }
  mithrilCertificate: ...
```

One Mithril certificate can anchor multiple CSMT proofs, which anchor multiple
credential proofs. The entire bundle is self-contained and independently
verifiable.
