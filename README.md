![ANTS](./brand/ants-lockup-h.svg)

> An open protocol for collective intelligence.

[Manifesto](./MANIFESTO.md) · [Governance](./GOVERNANCE.md) · [RFCs](./spec/) · [Issues](https://github.com/Ants-Community/ants/issues) · [Chat Space](https://matrix.to/#/#ants-community:matrix.org) · [ants.community](https://ants.community)

---

## What is ANTS

ANTS is a peer-to-peer protocol for running artificial intelligence as a shared, sovereign, verifiable substrate — a transport layer on which any honest economy can operate.

No central provider. No protocol-level token. No accounts. Every machine that joins the colony — a smartphone, a laptop, a workstation, a datacenter — both contributes to the network and draws from it. Communities organize their work through barter; commercial operators may serve under their own pricing; gift-economy peers serve for free. All four coexist on the same protocol, isolated from each other economically, interoperable at the transport layer.

The result is a single, distributed cognitive substrate: many models, many specialisations, many hardware tiers, many economic models, one protocol.

This repository is where that protocol is being designed in the open.

## Why it matters

Today, frontier artificial intelligence lives inside a handful of companies, in a handful of datacenters, under a handful of jurisdictions. This is the largest concentration of cognitive infrastructure in human history, and it has no precedent we can learn from.

ANTS is the proposition that intelligence, like the internet itself, can be a public protocol rather than a private product. That a colony of millions of machines — running open models, running proprietary models, running fine-tunes nobody has heard of yet — coordinated by good rules, can produce something no single machine could, including the biggest ones.

Read the [Manifesto](./MANIFESTO.md) for the longer answer.

## Status

**Day zero.** The protocol is being specified. There is no code in this repository yet — only the manifesto, the governance document, the brand, and the design RFCs that will become the architecture.

If you are looking for production software, come back later. If you want to help write what comes next, you are exactly on time.

## Architecture (in one breath)

ANTS is structured as three architectural layers, each with its own primitive:

1. **Transport** — peer-to-peer exchange of queries, responses, attestations, and cache lookups in BitTorrent-style sessions. Routing is via DHT; sessions are choke/unchoke negotiated per pair. This is the layer every peer speaks.

2. **Memory** — the [semantic cache](./spec/RFC-0002-semantic-cache.md), a distributed content-addressable store keyed by prompt embedding. Every honest answer the colony has ever given lives here, signed by its producer, retrievable by similarity, durable across years. The cache turns the colony's collective output into reusable infrastructure.

3. **Reputation** — a small, slow blockchain recording only misbehaviour facts, with a novel [Proof-of-Unique-Hardware](./spec/RFC-0004-reputation-pouh.md) consensus mechanism. One attested CPU equals one validator voice. No tokens, no stake, no plutocracy.

On top of these three layers, multiple economies coexist:

- The **community economy** — symmetric barter in normalised compute-seconds (NCS), no money, no token. See [RFC-0001](./spec/RFC-0001-community-economy.md).
- **Commercial economies** — operators serving for fiat payment under their own terms. See [RFC-0006](./spec/RFC-0006-payment-terms.md).
- **Gift economy** — peers who serve for free, no settlement.
- **Hybrid peers** — mode-switched per query.

Identity across all of these is [hardware-attested](./spec/RFC-0005-identity.md): one CPU, one peer. Verification follows a [three-tier model](./spec/RFC-0003-verification.md) that scales cost with the stakes of each query.

The formal specification of each layer lives under [`spec/`](./spec/).

## How to participate

This is day zero. The most valuable thing right now is sharp thinking, not code.

- **Open an issue** to debate any part of the design — the more concrete the critique, the better. Current open questions are tagged [`rfc-0001`](https://github.com/Ants-Community/ants/labels/rfc-0001) onwards.
- **Read the [Manifesto](./MANIFESTO.md) and [Governance](./GOVERNANCE.md)** and tell us where you disagree.
- **Watch this repository** to follow as RFCs are added and revised.
- **Join the [Matrix Space](https://matrix.to/#/#ants-community:matrix.org)** for asynchronous discussion in the `#General` room.

When the specification stabilises, working repositories for the reference node, the SDKs, and the verification toolchain will be created as sibling repos under [Ants-Community](https://github.com/Ants-Community).

## License

- Code and specifications in this repository are licensed under [Apache License 2.0](./LICENSE).
- The [Manifesto](./MANIFESTO.md) and [Governance](./GOVERNANCE.md) are released into the public domain under [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/), as declared in their footers.
- Future reference implementations of network-facing services will be released under [AGPL-3.0](https://www.gnu.org/licenses/agpl-3.0.html) in dedicated repositories, to prevent closed re-hosting of the network.

## A note on what ANTS is not

ANTS is not a company. ANTS does not issue a token. ANTS is not a model. ANTS is not a product.

ANTS is a protocol that carries any honest economy. The colony has no center. You are the center.
