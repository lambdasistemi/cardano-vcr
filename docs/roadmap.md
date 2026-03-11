# Roadmap

## Current state

Cardano VCR is in the **design phase**. The underlying MPFS infrastructure is
functional; the VCR protocol layer is being specified.

### Available components

| Component | Status | Description |
|-----------|--------|-------------|
| [cardano-mpfs-onchain][onchain] | Working | Cage validators (Aiken, Plutus V3) |
| [cardano-mpfs-offchain][offchain] | Phase 4 complete | Indexer, TxBuilder, RocksDB state |
| [haskell-mts][mts] | Working | CSMT + MPF trie implementations |
| [cardano-utxo-csmt][csmt] | Working | UTxO set Merkle tree, HTTP API |

### Gaps

| Gap | Description | Blocking |
|-----|-------------|----------|
| MPFS HTTP API | Phase 5 of cardano-mpfs-offchain (Servant API, Docker deployment) | User-facing credential operations |
| VCR protocol | This repository — schema/credential encoding, proof bundle format | Everything above MPFS |

## Phase 1: Protocol specification

- Define schema encoding format (compatible with W3C `credentialSchema`)
- Define credential encoding format (all W3C required + optional fields)
- Define proof bundle format (for off-chain verification)
- Define verifiable presentation format (multi-credential bundles)
- Document resolver interface

## Phase 2: Reference implementation

- Schema authority cage configuration (append-only policy)
- Credential issuer cage configuration
- TxBuilder extensions for VCR-specific operations
- Proof bundle serialization (CBOR)

## Phase 3: Verification SDKs

- On-chain verification library (Aiken helper functions for Plutus validators)
- Off-chain verification library (Haskell, for proof bundle verification)
- Lightweight verification library (for mobile/browser, minimal dependencies)

## Phase 4: Tooling and deployment

- CLI for schema registration and credential operations
- HTTP API for credential issuance, revocation, and verification
- Docker deployment (plutimus.com)
- Documentation and tutorials

[onchain]: https://github.com/cardano-foundation/cardano-mpfs-onchain
[offchain]: https://github.com/paolino/cardano-mpfs-offchain
[mts]: https://github.com/paolino/haskell-mts
[csmt]: https://github.com/cardano-foundation/cardano-utxo-csmt
