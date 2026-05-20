![ANTS](./brand/ants-lockup-h.svg)

> An open protocol for collective intelligence.

[Manifesto](./MANIFESTO.md) · [Governance](./GOVERNANCE.md) · [Governance](./GOVERNANCE.md) · [Issues](https://github.com/Ants-Community/ants/issues) · [Chat Space](https://matrix.to/#/#ants-community:matrix.org) · [ants.community](https://ants.community)

---

## What is ANTS

ANTS is a peer-to-peer protocol for running artificial intelligence as a shared, sovereign, verifiable network — owned by no one, accessible to everyone who participates.

No central provider. No token. No accounts. Every machine that joins the colony — a laptop, a phone, a workstation, a datacenter — both contributes to the network and draws from it, in proportion to what it gives.

The result is a single, distributed cognitive substrate: many models, many specialisations, many hardware tiers, one protocol.

This repository is where that protocol is being designed in the open.

## Why it matters

Today, frontier artificial intelligence lives inside a handful of companies, in a handful of datacenters, under a handful of jurisdictions. This is the largest concentration of cognitive infrastructure in human history, and it has no precedent we can learn from.

ANTS is the proposition that intelligence, like the internet itself, can be a public protocol rather than a private product. That a colony of millions of imperfect machines, coordinated by good rules, can produce something no single machine could — including the biggest ones.

Read the [Manifesto](./MANIFESTO.md) · [Governance](./GOVERNANCE.md) · [Governance](./GOVERNANCE.md) for the longer answer.

## Status

**Day zero.** The protocol is being specified. There is no code in this repository yet — only the manifesto and the design documents that will become the architecture.

If you are looking for production software, come back later. If you want to help write what comes next, you are exactly on time.

## Architecture (in one breath)

ANTS is structured as six interoperating layers:

1. **Economy** — BitTorrent-style symmetric barter. To consume, contribute. The currency is participation itself, never a token.
2. **Roles** — each peer chooses what it offers: inference, verification, caching, storage, routing, annotation. Roles are first-class; a laptop without a GPU is as legitimate a node as a server with eight H100s.
3. **Verification** — every peer's code runs inside a Trusted Execution Environment and produces a hardware-signed attestation. Cheating is not punished by reputation — it is structurally prevented.
4. **Reputation** — an append-only public ledger records peer behaviour (slashing, agreements, disputes). Used to weight trust, never to settle finance.
5. **Identity** — proof of unique hardware. One CPU, one identity. No KYC. Sybil attacks must rent thousands of physical chips, not register thousands of accounts.
6. **Cold start** — new peers without specialised models join as routers, verifiers, and cache nodes, earning the right to consume inference by contributing the layers they can.

A formal specification of each layer will live under `spec/` as RFCs, once the directory exists.

## How to participate

This is day zero. The most valuable thing right now is sharp thinking, not code.

- **Open an issue** to debate any part of the design — the more concrete the critique, the better.
- **Read the manifesto** and tell us where you disagree.
- **Watch this repository** to follow as RFCs are added.
- **Join Matrix** `#ants-community:matrix.org` for asynchronous discussion (`#General` room).

When the specification stabilises, working repositories for the reference node, the SDKs, and the verification toolchain will be created as sibling repos under [Ants-Community](https://github.com/Ants-Community).

## License

- Code and specifications in this repository are licensed under [Apache License 2.0](./LICENSE).
- The [Manifesto](./MANIFESTO.md) · [Governance](./GOVERNANCE.md) · [Governance](./GOVERNANCE.md) is released into the public domain under [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/), as declared in its footer.
- Future reference implementations of network-facing services will be released under [AGPL-3.0](https://www.gnu.org/licenses/agpl-3.0.html) in dedicated repositories, to prevent closed re-hosting of the network.

## A note on what ANTS is not

ANTS is not a company. ANTS is not a token. ANTS is not a model. ANTS is not a product.

ANTS is a colony. The colony has no center. You are the center.
