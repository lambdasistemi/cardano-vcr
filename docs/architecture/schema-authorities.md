# Schema Authorities

Schema authorities manage registries of [credential schemas][w3c-schemas]. Each
authority operates an independent MPFS cage with its own governance policy,
functioning as a [verifiable data registry][w3c-registry] in W3C terms.

## Why multiple authorities?

A single global schema registry controlled by one oracle would be a trust
bottleneck. If the oracle can modify or delete schemas, every credential issued
under those schemas is retroactively undermined. Consider:

1. Authority A registers schema "UniversityDegree"
2. University X issues 10,000 credentials referencing that schema
3. Authority A deletes the schema
4. All 10,000 credentials are now unverifiable against the schema registry

With multiple independent authorities, this risk is contained. Authority A's
deletion only affects credentials that reference Authority A's schemas. Schemas
registered with Authority B are untouched.

## Schema data model

### Key

```
key = blake2b_256(schema_definition)
```

The key is the hash of the schema definition itself. This guarantees uniqueness:
two identical schemas produce the same key, preventing duplicates within a single
authority's registry.

### Value

```
{ definition   : ByteArray        -- Schema structure (field names + types)
, resolver     : Maybe ScriptHash -- Optional Plutus validator for custom logic
, revocable    : Bool             -- Can credentials under this schema be revoked?
, creator      : PubKeyHash       -- Who registered this schema
}
```

| Field | Purpose |
|-------|---------|
| `definition` | The schema structure, defining what fields a credential must contain and their types. Analogous to a JSON Schema or Solidity ABI type string. |
| `resolver` | Optional reference to a Plutus validator that must be satisfied when issuing credentials under this schema. Enables custom logic such as payment requirements or access control. |
| `revocable` | Whether credentials issued under this schema can be revoked (deleted from the issuer's trie). Some credential types (e.g. academic degrees) may be permanently non-revocable. |
| `creator` | The public key hash of the entity that registered the schema. Informational — does not grant special privileges. |

## Authority policies

Each authority sets its own cage policy. Common configurations:

### Append-only (recommended for most authorities)

- **Insert**: Allowed (anyone can request schema registration)
- **Delete**: Forbidden at the validator level
- **Update**: Forbidden at the validator level

Once a schema is registered, it exists permanently. This is the strongest
guarantee for credential issuers and holders.

### Governed

- **Insert**: Allowed with governance approval
- **Delete**: Allowed under exceptional circumstances (e.g. schema found to be
  defective)
- **Update**: Forbidden (register a new schema version instead)

Appropriate for consortium-managed authorities where a governance body oversees
the registry.

### Open

- **Insert**: Allowed
- **Delete**: Allowed by creator
- **Update**: Forbidden

Appropriate for experimental or research contexts where schemas may be
provisional.

## Trust model

A verifier decides which schema authorities it recognizes. This is a
configuration choice, not a protocol constraint.

Example trust policies:

- "I accept any credential whose schema exists in the government standards
  authority (cage token `abc123`)"
- "I accept credentials from any issuer, but only if the schema is registered
  with one of these three authorities: `[abc, def, ghi]`"
- "I accept any schema from any authority" (permissive)

The protocol provides the cryptographic proof that a schema exists in a
specific authority's registry. The policy layer decides whether that authority
is trusted.

## Schema verification

To verify that a credential's schema is valid:

1. Extract the schema reference from the credential value (authority cage
   token ID + schema key)
2. Obtain the authority's cage root (via reference input or off-chain query)
3. Verify the Merkle proof that the schema key exists in the authority's trie
4. Optionally compare the schema definition against expected structure

This is the same proof mechanism used for credentials — consistent across the
entire protocol.

[w3c-schemas]: https://www.w3.org/TR/vc-data-model-2.0/#data-schemas
[w3c-registry]: https://www.w3.org/TR/vc-data-model-2.0/#dfn-verifiable-data-registries
