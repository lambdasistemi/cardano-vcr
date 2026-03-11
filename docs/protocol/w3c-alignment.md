# W3C Alignment

Cardano VCR implements the [W3C Verifiable Credentials Data Model 2.0][w3c-vc],
a W3C Recommendation published May 15, 2025. This page maps every normative
concept from the specification to its Cardano VCR counterpart.

## Ecosystem roles

The W3C specification defines four roles:

| W3C Role | Definition | Cardano VCR |
|----------|-----------|-------------|
| **Issuer** | Asserts claims, creates credentials, transmits to holder | Credential cage oracle (cage token owner) |
| **Holder** | Possesses credentials, creates presentations | Wallet receiving Merkle proofs |
| **Verifier** | Receives and processes credentials/presentations | Plutus validator or off-chain entity |
| **Verifiable Data Registry** | Mediates creation/verification of identifiers, schemas, revocation registries | MPFS cages (both schema and credential) |

The specification states:

> "A role an entity can perform by asserting claims about one or more subjects,
> creating a verifiable credential from these claims, and transmitting it to a
> holder."

In Cardano VCR, the issuer is the cage oracle — the entity that controls the
credential trie. The cage token's on-chain identity cryptographically binds the
issuer to their credentials.

## Credential data model

The W3C specification defines required and optional properties for verifiable
credentials. Here is how each maps to Cardano VCR:

### Required properties

| W3C Property | W3C Requirement | Cardano VCR |
|-------------|----------------|-------------|
| `@context` | Must include `https://www.w3.org/ns/credentials/v2` | Encoded in credential value; verifiers know the context from the protocol |
| `type` | Must include `VerifiableCredential` | Implicit — all trie entries under a credential cage are credentials |
| `issuer` | Entity asserting claims | Cage token ID identifies the issuer |
| `credentialSubject` | Entity about which claims are made | Part of the credential trie value |
| `validFrom` | Issuance timestamp | `time` field in the credential value |

### Optional properties

| W3C Property | Cardano VCR |
|-------------|-------------|
| `id` | Trie key = `blake2b(schema_uid ++ recipient ++ nonce)` |
| `validUntil` | `expiration` field in credential value |
| `credentialSchema` | Reference to schema authority cage token ID + schema key |
| `credentialStatus` | **Trie membership IS the status** — present = valid, absent = revoked |
| `evidence` | Can be included in value or referenced off-chain |
| `proof` | Merkle proof against the cage root |
| `refreshService` | Not applicable (trie is always current) |
| `termsOfUse` | Can be encoded in schema or credential metadata |

### The `credentialStatus` insight

The W3C specification says:

> "When an issuer desires to enable status information for a verifiable
> credential, they MAY add a `credentialStatus` property."

The specification mentions [Bitstring Status List][bitstring] as one mechanism.
Cardano VCR uses a fundamentally different approach: **trie membership IS the
status**. A credential exists in the trie (valid) or it does not (revoked).

This has two advantages over status lists:

1. **Proof of revocation** — you can prove a credential has been revoked by
   providing a proof of non-membership against the current root
2. **True deletion** — revoked credentials are removed from the trie, not just
   flagged. The trie gets smaller, not larger, as credentials are revoked

## Verifiable presentations

The specification defines:

> "Data derived from one or more verifiable credentials issued by one or more
> issuers that is shared with a specific verifier."

In Cardano VCR, a verifiable presentation is a **Cardano transaction** containing:

- Multiple **reference inputs** (one per issuer cage, providing the credential
  roots)
- Multiple **Merkle proofs** in the transaction redeemer (one per credential
  being presented)

A single transaction can verify credentials from N different issuers. This is
a native verifiable presentation.

For off-chain presentations, a holder bundles multiple Merkle proofs into a
proof package that a verifier processes without blockchain interaction.

## Trust model

The specification states:

> "Verifiability of a credential does not imply the truth of claims encoded
> therein."

And:

> "Verifiers apply their own policies."

Cardano VCR embodies this exactly. The protocol provides cryptographic
verification (the credential exists, was issued by a specific issuer, has not
been revoked). Whether the verifier **trusts** that issuer is a policy decision
outside the protocol.

A verifier (Plutus validator or off-chain service) decides:

1. Which **schema authorities** it recognizes
2. Which **credential issuers** it trusts for specific schemas
3. What **additional checks** to perform (expiration, evidence, etc.)

## Verifiable Data Registry

The specification defines:

> "A system performing a role by mediating creation and verification of
> identifiers, verification material, schemas, revocation registries, and other
> relevant data."

MPFS cages are verifiable data registries. Schema cages mediate schema creation
and verification. Credential cages mediate credential issuance and revocation.
The trie root, available as a Cardano UTxO, is the verifiable anchor for all
data in the registry.

## Securing mechanisms

The specification requires:

> "A conforming document MUST be secured by at least one securing mechanism."

Two approaches are defined:

1. **Embedded proofs** — signatures within the credential
2. **Enveloping proofs** — signatures wrapping the credential

Cardano VCR uses a third approach enabled by Merkle trees: **inclusion proofs**.
The credential is secured by its inclusion in a Merkle trie whose root is
anchored on-chain (in a UTxO controlled by the issuer's cage validator). The
proof is compact (O(log n) hash steps) and independently verifiable.

This is compatible with the W3C model because the specification is deliberately
open about securing mechanisms — it defines requirements (tamper-evidence,
verifiability) rather than prescribing specific cryptographic schemes.

## Conformance

### Conforming issuer

> "Produces conforming documents, MUST include all required properties, and MUST
> secure documents using a securing mechanism."

Credential cage oracles produce credentials with all required W3C properties,
secured by Merkle inclusion in a trie whose root is on-chain.

### Conforming verifier

> "Consumes conforming documents, MUST perform verification, MUST check required
> properties, and MUST produce errors when non-conforming documents are detected."

On-chain verifiers (Plutus validators) verify Merkle proofs against cage roots
via reference inputs. Off-chain verifiers verify proof bundles against trusted
roots. Both reject invalid proofs.

[w3c-vc]: https://www.w3.org/TR/vc-data-model-2.0/
[bitstring]: https://w3c.github.io/vc-bitstring-status-list/
