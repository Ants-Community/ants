# ANTS RFCs

Design documents for the ANTS protocol. Each RFC is a draft, written to be argued with.

The protocol is structured as a stack of layers. Reading order matches dependency order.

| Number | Topic | Status | Depends on |
|--------|-------|--------|------------|
| [RFC-0001](./RFC-0001-community-economy.md) | Community Layer Economics — barter, the local ledger, choke/unchoke, the specialist credit problem | Draft v0.3 | RFC-0002, RFC-0006 |
| [RFC-0002](./RFC-0002-semantic-cache.md) | Semantic Cache Layer — distributed memory; canonical embedding model governance | Draft v0.2 | RFC-0008, RFC-0009 |
| [RFC-0003](./RFC-0003-verification.md) | Verification — three tiers, unified by commit-at-send + anytime-valid e-process; closed-form deterrence threshold | Draft v0.2 | RFC-0001, RFC-0002, RFC-0004, RFC-0005 |
| [RFC-0004](./RFC-0004-reputation-pouh.md) | Reputation — consensus-free CRDT for slashing + PoUH chain as ordered witness + A-as-bond for high-stakes acts | Draft v0.3 | RFC-0003, RFC-0005 |
| [RFC-0005](./RFC-0005-identity.md) | Identity — hardware attestation, multi-vendor TEE, Sybil economics | Draft v0.1 (early) | — |
| [RFC-0006](./RFC-0006-payment-terms.md) | Payment Terms & Economic Pluralism — how multiple economies coexist | Draft v0.1 (early) | RFC-0001 |
| RFC-0007 | Post-v1.0 Governance — TSC mechanics, transition from BDFL | Planned | [GOVERNANCE.md](../GOVERNANCE.md) |
| [RFC-0008](./RFC-0008-wire-formats.md) | Wire Formats, Crypto Primitives, Reference Constants — the implementation glue; drand failover, BLS/Ed25519 K-transition | Reference v0.2 | — |
| [RFC-0009](./RFC-0009-canonical-numerics.md) | Canonical Numerics for Verifiable Inference — INT8-GPTQ default + FP16 fallback; Y_canon unifies serving/caching | Draft v0.2 (early) | RFC-0002, RFC-0003, RFC-0008 |

## Status legend

- **Draft** — the RFC is technically complete enough to be implemented experimentally; specifics may still change.
- **Draft (early)** — the RFC defines the design intent and the major open questions; specifics are explicitly under-specified and expected to be revised after testnet experience.
- **Planned** — the RFC has a designated number and topic; the document has not yet been drafted.
- **Reference** — a living-standard technical specification that is prescriptive rather than exploratory; versioned independently from the design RFCs and updated when a primitive must change.

## Reading order

If you are new to the project and want to understand the architecture, read in this order:

1. [`../MANIFESTO.md`](../MANIFESTO.md) — the philosophical grounding (5 minutes)
2. [`../GOVERNANCE.md`](../GOVERNANCE.md) — who decides, and until when (10 minutes)
3. **RFC-0001** — the community economy, the layer most other things rest on
4. **RFC-0002** — the semantic cache, the architectural keystone
5. The rest in numerical order — verification, reputation, identity, payment terms

## How to contribute

Open an issue on this repository tagged with the relevant RFC number (e.g., `rfc-0002`). Typo and clarity fixes can be submitted as pull requests directly; substantive design questions should start in an issue.

All RFCs are released under [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/). Translate them. Quote them. Disagree with them in public. Fork them.
