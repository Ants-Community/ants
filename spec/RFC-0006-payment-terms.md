# RFC-0006 — Payment Terms & Economic Pluralism

**Status:** Draft · v0.1 (early — substantial revision expected)
**Topic:** How peers operating under different economic models — barter, fiat, gift, subscription, hybrid — coexist on the same protocol without forcing each other to converge.
**Audience:** You, if you have wondered how Anthropic or OpenAI could ever participate in a project whose community doesn't accept money.

---

## What this document specifies

The ANTS protocol carries queries, responses, attestations, and cache lookups. It does not specify how peers settle with each other economically. That is left to the peers.

This RFC specifies the **mechanism** by which peers declare their economic terms — the `payment_terms` metadata field that accompanies every query and every advertised service — and the **rules** that govern how peers operating under different terms can interoperate, decline, or transact.

The result is a protocol on which:

- The ANTS community runs barter (NCS, RFC-0001).
- Commercial laboratories may serve their proprietary models for fiat payment.
- Gift-economy peers may serve for free.
- Hybrid peers may switch modes per query.
- All four populations are interoperable at the transport layer, isolated at the economic layer, and free to refuse each other when terms do not align.

---

## Why this exists

The architectural rationale was made in the Manifesto (v0.2 Thesis 6 and 15) and is restated here for completeness.

Centralised AI today is concentrated in three or four datacentres owned by three or four corporations. The ANTS project's stated goal is to provide an open protocol that distributes intelligence across millions of machines. To achieve this, the protocol must be *attractive* to participants of every economic disposition — not just to those willing to operate by barter.

A protocol that demands money is captured by money. A protocol that forbids money has limited adoption by anyone whose livelihood depends on AI revenue. The protocol that wins is the one that takes no position on payment, leaving that to peers.

This is the same architectural choice that made HTTP win against AOL, CompuServe, and Minitel: HTTP did not know what payment was. It simply transmitted bytes. Every business model that wanted to exist on the web could exist on HTTP, including the gift economy (Wikipedia) and the most aggressive commerce (Amazon). HTTP did not have to choose.

ANTS does not have to choose.

---

## The payment_terms field

Every request and every advertised service on the ANTS protocol carries a `payment_terms` field. The field is a structured object describing the *acceptable* settlement model for that party.

Schema (informal):

```
payment_terms {
  schemes: [PaymentScheme]   # one or more acceptable schemes
  validity: timestamp        # how long these terms are honoured
  scope: enum                # "per_query" | "session" | "subscription"
  notes: string              # human-readable additional terms
}

PaymentScheme = one of {
  none {}                                          # free service, no settlement
  ncs { rate_table: ref }                         # ANTS community barter, RFC-0001
  fiat_stripe { currency, rate_per_unit, key }    # commercial; payment via Stripe link
  fiat_lightning { sats_per_unit, invoice_addr }  # commercial; Lightning Network payment
  subscription { provider, plan_id }              # subscription-based access
  custom { schema_uri }                            # any custom scheme, externally specified
}
```

When a consumer client sends a query, it includes its *acceptable* payment_terms — the schemes it will agree to pay under. When a server peer advertises a service, it declares its *offered* payment_terms — what it will accept.

A match is made when at least one of the consumer's acceptable schemes intersects with the server's offered schemes. If no match exists, no transaction occurs. The two parties move on.

---

## Standard schemes

The protocol ships with five named schemes, fully specified. Custom schemes may be defined by external standards but are not protocol-blessed.

**`none`** — the gift economy. The server commits to serving without expectation of return. Quality and availability are at the server's discretion; the consumer accepts whatever is offered. Reputation still accumulates (gifted work is verified like any other, contributes to the producer's quality score).

**`ncs`** — community barter, as specified in RFC-0001. The server expects credit in the local barter ledger; the consumer commits to honouring the rate table. NCS-only peers will not transact with peers offering only fiat schemes.

**`fiat_stripe`** — commercial fiat payment via Stripe. The server specifies a per-unit rate (e.g., per-token, per-cache-hit, per-second-of-inference) in a named currency, and a Stripe payment endpoint to which the consumer's payment is directed. Stripe handles settlement; the protocol routes the data.

**`fiat_lightning`** — commercial fiat payment via the Bitcoin Lightning Network. The server specifies a per-unit rate in satoshis and an invoice address. Suitable for micro-payments (Lightning's strength) where Stripe-style settlement would be too heavy. Examples: a peer charges 50 sats per token for a niche fine-tuned model.

**`subscription`** — periodic payment for unlimited or rate-limited access. The consumer's client maintains a valid subscription credential (issued out-of-band by the provider); each query carries that credential; the server validates and serves. The protocol does not handle subscription billing — that is the provider's concern.

---

## How peers discover each other's terms

Peers advertise their offered payment_terms as part of their service registration in the DHT. A consumer client searching for "a peer that can serve Llama-3.3-70B inference" can filter the candidate set by acceptable payment terms before making any actual query.

The DHT query returns metadata only — the consumer learns *which* peers will serve and *under what terms*, without committing to a transaction. The consumer's client then ranks candidates by a composite of:

- Acceptable payment terms (a hard filter)
- Estimated latency (network distance)
- Producer reputation (from RFC-0003 and RFC-0004)
- Verification tier offered
- Cached response availability (if applicable)

Only after this ranking does the consumer actually transmit the query and its payment.

---

## Cross-economy cache settlement

The cache layer (RFC-0002) creates a non-trivial settlement problem: what happens when a cache entry produced under one economy is served to a consumer operating under another?

Concrete scenario: a community-layer peer produces a high-quality answer about Italian construction law, signed and cached. Six months later, a commercial peer's customer (paying fiat) issues a similar query. The cache returns the community peer's answer. The commercial peer serves it to the paying customer. What does the community peer earn?

The protocol's current answer:

- **The cache entry carries the original producer's payment_terms,** declared at write time. The community peer's entry declares `ncs` only.
- **The cache host (the peer responding to the lookup)** declares its own payment_terms for hosting the lookup.
- **When the commercial peer serves the cached answer to the paying customer**, the commercial peer pays:
  - The cache host: in whatever the host accepts (typically NCS or fiat, declared by host).
  - The original producer: in the producer's declared terms (`ncs` in this scenario).
  - If the commercial peer cannot pay in NCS (e.g., it has no NCS balance), it offers a fiat-to-NCS conversion through one of the **community-layer fiat gateways** — a small set of peers that voluntarily convert between fiat and NCS at posted rates, for a small spread.

The community-layer fiat gateway is *not* protocol-mandated. It is an emergent role that the community may or may not develop. If it does not, commercial peers cannot serve community-cached responses to paying customers — they must request fresh inference instead. This is acceptable; it is not the protocol's job to force a fiat-to-NCS bridge to exist.

---

## Isolation between economies

Several properties are guaranteed by protocol design:

**Cross-economy reputation does not leak.** A peer's reputation in the community-layer ledger is separate from its reputation among commercial customers. A peer that performs honestly for commercial customers does not automatically inherit standing in the community ledger, and vice versa. Each economy maintains its own quality signal.

**No peer can be forced into an economy.** A peer offering only `ncs` may decline any commercial query. A peer offering only `fiat_stripe` may decline any community-layer barter query. The protocol routes around the refusal.

**Sybil resistance is shared.** Hardware attestation (RFC-0005) is universal across economies. A peer cannot create separate identities for community and commercial operations to avoid reputation consequences. One CPU, one identity, whatever economy that identity participates in.

**No protocol-level rent extraction.** The protocol does not take a percentage of any payment. The community foundation does not take a cut of commercial transactions. All settlement is bilateral between consumer and server, mediated only by the payment scheme they have agreed on.

---

## What this means for OpenAI, Anthropic, and the rest

A practical scenario, for clarity.

**Anthropic decides to participate in ANTS.** They set up one or more peers running Claude, attested through the standard hardware attestation path. They declare their payment_terms as `fiat_stripe { currency: USD, rate_per_unit: $0.003 per 1K input tokens, $0.015 per 1K output tokens }`. They advertise their service in the DHT.

ANTS users running community-layer clients see Anthropic's peers in the discovery results, alongside community peers serving open-weight models. The community user's client filters by acceptable payment terms: if the user has not configured fiat acceptance, Anthropic's peers are filtered out. If the user has configured fiat acceptance (with a Stripe-linked card), Anthropic's peers appear as commercial options at the stated price.

The user, query by query, decides whether to send to the community (free, via barter the user has earned) or to Anthropic (paid, but with frontier-quality output). Anthropic earns its usual revenue. The community earns nothing from this query. The protocol earns nothing.

Anthropic's queries, like all others, may produce cached entries — which become available to subsequent queries (under the cross-economy settlement rules above). Anthropic's reputation accumulates separately from the community-layer reputation. Their cached entries flag `producer_economy: commercial`, allowing community peers to choose whether to retrieve them.

This is a workable arrangement. Anthropic gains distribution at minimal additional cost. The community gains access to frontier intelligence for users willing to pay. The protocol stays neutral.

---

## What we have not figured out yet

Significant open problems:

- **Fiat-to-NCS gateway design.** The protocol describes the gateway as "emergent." But for it to actually emerge, somebody has to do it first, at some risk. Are there bootstrap mechanisms? Probably yes — a foundation-coordinated initial gateway with strict per-peer caps, transitioning to community-operated.
- **Subscription model details.** How exactly does the subscription credential get verified per-query without revealing the consumer's identity? Likely via blinded credentials, but the cryptography is non-trivial.
- **Pricing manipulation.** A commercial peer could collude with a cache host to inflate cache-hit charges. Mitigation: cache-hit prices are bounded by protocol-declared parameters, not by host-declared.
- **Subscription pricing vs barter exchange rate.** Should a community-layer peer be able to "convert" their accumulated NCS into a discount on a commercial subscription? Currently we say no. This is a design choice that should be revisited.
- **Currency volatility.** A `fiat_lightning` rate quoted in sats becomes wildly different in USD over time as Bitcoin moves. The protocol should support either (a) dynamic re-quoting on a per-query basis, or (b) brief validity windows on price quotes. Currently unspecified.
- **Tax and reporting.** Commercial peers operating across jurisdictions must handle their own tax compliance. The protocol is silent on this; the protocol cannot solve it; the protocol should not pretend to.
- **Customer-paying-customer routing.** A consumer who has paid Anthropic for inference may want to forward the answer to a friend. Is that re-distribution covered by the original payment? This is a licensing question, not a protocol question; but the protocol should not technically prevent it, and operators should make their terms explicit in their `notes` field.

---

## Adversarial considerations

**Payment terms spoofing.** A peer claims to offer service for `ncs` but secretly demands fiat after the work is done. Defence: payment_terms are signed by the peer's attested identity at advertisement time and at transaction time. Mismatch is provable misbehaviour, reported to the reputation chain.

**Price-collusion rings.** A group of commercial peers agree to keep prices high. The protocol cannot prevent this; antitrust law in the operator's jurisdiction handles it. The protocol does not interfere either way.

**Predatory pricing.** A well-resourced peer prices below cost to drive out competition. Same answer as above: not the protocol's problem.

**Subscription-credential theft.** A consumer's subscription credential is stolen, used by attackers. Mitigation: each subscription provider designs their own credential robustness. The protocol's role is only to forward the credential to the provider for validation.

---

## How to contribute to this document

Open an issue on `Ants-Community/ants` with the tag `rfc-0006`. Particularly valuable: practical bootstrap proposals for the fiat-NCS gateway, formal analysis of cross-economy cache settlement, alternative payment scheme designs we have not considered.

The document is CC0.

---

*Drafted by the ANTS founding contributors, May 2026.
The protocol does not choose between economies. The peers do.
*
