# RFC-0002 — Semantic Cache Layer

**Status:** Draft · v0.2
**Topic:** The distributed memory of the network. How peers retain, address, and reuse every honest answer the colony has ever given.
**Audience:** You, if you are willing to disagree in writing.

---

## A second Tuesday

It is still Tuesday evening. A different laptop, in Buenos Aires.

A student asks a question about the difference between *causa* and *contraprestación* in Argentine contract law. Her client computes an embedding of the question in 14 milliseconds and queries the network. Forty milliseconds later, the cache returns three candidate answers — each one given in the past by a different peer, each one signed and attested, each one accompanied by a counter showing how many times it has been retrieved and how many thumbs-up it has accumulated.

The top candidate was originally produced eight months ago by a peer in La Plata, an attorney who specialises in commercial contract law. She has earned a small fraction of NCS every time her answer has been retrieved since — fourteen hundred and twelve times so far. She has never met the student in Buenos Aires. She probably never will.

The student's client serves the cached answer in 80 milliseconds total, end to end. The student gets a near-instant, expert-grade answer at zero marginal cost. The attorney earns her share of NCS automatically. The network spends no fresh GPU time. The cache shard hosting that embedding region — held redundantly across nine peers in five jurisdictions — earns its hosting fee.

This is the second Tuesday. It will happen ten thousand times tonight, and a hundred thousand tomorrow.

This document specifies how.

---

## Why this layer exists

Queries are infinite. Good answers are not.

This is the empirical observation on which the entire ANTS economy comes to rest. The space of possible questions a user could ask an AI is effectively unbounded — every concept, every domain, every variation in phrasing, every personal context. But the space of *high-quality canonical answers* is enormously smaller. Most questions, when reduced to their semantic essence, are variants of questions that have been asked and answered before. The same question phrased in three languages, at three levels of formality, with three different motivating contexts, has — to a useful approximation — the same correct answer.

A network that rediscovers this answer with a fresh inference each time wastes compute, money, and electricity. A network that retains it, can find it by similarity, and serves it again gets *exponentially* more efficient per query as it ages. The cache is not an optimisation that makes ANTS faster. It is the mechanism that makes ANTS *possible at all*, given the cost of inference in 2026.

Without this layer, every query costs a peer's GPU-second somewhere on the network. With this layer, the *vast majority* of queries cost a cache lookup — three to four orders of magnitude cheaper. The network capacity, measured in queries served per real GPU-second of underlying hardware, multiplies by 100× to 1000× depending on how aggressively the cache is used.

This is also the layer that resolves the *specialist credit problem* in the community economy. A specialist whose deep expertise produces an exceptional answer is no longer paid once and forgotten. The answer lives. The answer earns. The specialist's value compounds. RFC-0001 assumes this resolution; this RFC delivers it.

---

## What the cache is

The cache is a **distributed, content-addressable, append-mostly key-value store**. The key is a fixed-length embedding vector. The value is a cached answer with its provenance, its attestation, and its history.

It is *not* a blockchain. There is no global consensus on cache state. There is no canonical ordering of writes. Two peers may legitimately hold slightly different snapshots of the same embedding region and resolve them differently for different queries; the network's quality emerges from the aggregate of these locally-resolved decisions over time.

It is *not* a centralised database. There is no master shard. No company runs the cache. The cache lives across the peers, sharded by embedding space, replicated for durability, and queried by anyone speaking the protocol.

It is *not* an immutable archive. Old entries decay. Out-of-date answers get invalidated. Low-quality entries get downweighted by feedback. The cache is a living substrate, not a frozen record.

Architecturally, the closest analogue is **IPFS** for content addressing and durability, combined with **vector databases** for the lookup semantics, on top of a **Kademlia-style DHT** for routing. None of these existed in the precise configuration we need. ANTS specifies the configuration.

---

## The cache entry

Each entry in the cache is a signed structured record of the following shape:

```
embedding:        f32[d]              # the embedding of the prompt
                                      # d is the model-defined dimension
embedding_model:  string              # e.g. "ants-embed-v1"
prompt_hash:      bytes[32]           # SHA-256 of the original prompt
response:         bytes                # the cached response payload
response_model:   string              # e.g. "llama-3.3-70b-instruct"
producer:         peer_id              # attested peer that generated the response
attestation:      bytes                # hardware-attested signature of the work
created:          timestamp
validity_class:   enum                # "ephemeral" | "weeks" | "months" | "years" | "perennial"
retrieval_count:  u64
positive_signals: u64                 # human thumbs-up
negative_signals: u64                 # human thumbs-down
challenge_count:  u32                 # times another peer has re-verified and confirmed
last_challenged:  timestamp
```

A few choices in this schema are worth flagging.

**The prompt itself is not stored, only its hash.** This is a privacy default. The embedding is the search key; the prompt is recoverable only by whoever already has it. (Privacy implications discussed below.)

**The response payload is opaque.** It might be text, structured JSON, an image, an audio file — the protocol does not interpret it. The model identifier tells the consumer what to expect; the consumer's client decodes accordingly.

**The validity class is producer-declared but verifier-revisable.** A response to "what year did the French revolution begin?" can declare `validity_class: perennial`. A response to "what is the EUR/USD rate?" must declare `validity_class: ephemeral` (a few minutes at most). Producers who systematically over-declare validity get downweighted by the verification system and eventually lose ability to write to the cache.

**Signals are append-only and signed.** Each positive or negative feedback is a separate signed event from a different attested peer. The cache entry's counters are summaries; the underlying events are retained for auditability.

---

### What the `response` field contains

*Clarified in v0.2.* The `response` field stores **the canonical-recipe
output `Y_canon`** as specified in
[RFC-0009 v0.2](./RFC-0009-canonical-numerics.md) §"What the network
commits to" — not whatever fast-path output the producer's internal
implementation may have computed alongside.

Every cache retrieval therefore returns byte-identical content for the
same entry regardless of which peer serves it. This is what makes Tier 2
audits of cache hits trivial: the auditor recomputes the canonical recipe
and matches against the cached `response` byte-for-byte, with the
honest-noise floor `σ` effectively zero under the integer-canonical
recipe.

Producers whose serving implementation produces `Y_canon`-equivalent
outputs natively pay no verifiability tax. Producers running a different
fast path must compute `Y_canon` separately, doubling their per-served
compute cost. The economic pressure toward canonical-recipe-as-production
is the convergence mechanism described in RFC-0009 v0.2; this RFC
inherits the consequence — every cache entry is canonical.

---

## The canonical embedding model

Coordination matters. If every peer used a different embedding model, queries and cached entries would live in incompatible vector spaces and the lookup would be impossible.

The protocol therefore specifies, at any given time, a **canonical embedding model**, versioned. Peers may advertise support for multiple versions; new versions are introduced through coordinated protocol upgrades.

**Current canonical model: `ants-embed-v1`**, derived from BAAI's BGE-M3 family — multilingual, dense+sparse, well-benchmarked for cross-lingual retrieval, open-weight, runs in under 200ms on commodity CPU. The protocol pins a specific checkpoint of BGE-M3 by SHA-256 of its weights so that all peers compute identical embeddings for identical inputs.

Future versions (`ants-embed-v2`, etc.) will be introduced when one of the following becomes true: a substantially better open-weight embedding model emerges; the current model is found vulnerable to embedding-poisoning attacks; or the field has moved past 1024-dimensional dense representations toward something more efficient.

A version transition is a coordinated event lasting six to twelve months: peers begin serving both `v1` and `v2` embeddings in parallel; cache entries get gradually re-embedded into `v2` by background workers; the network monitors lookup quality on both; `v1` is deprecated when `v2` adoption exceeds 95% by traffic. There is no flag day.

The choice of canonical model is the single most consequential governance decision the cache layer makes. It is therefore made through the full RFC process, not by the BDFL alone, even in the v0.x period.

---

## Governance of the canonical embedding model

*Added in v0.2 to close B5 from the pre-implementation criticality
review.*

The canonical embedding model is the single most consequential governance
decision the cache layer makes — v0.1 of this RFC said so but did not
specify the process. It is the one protocol-level coordination object
that every peer must agree on bit-for-bit, a property no other component
of ANTS requires. It therefore warrants governance rules above the
standard amendment process.

### Proposal

A proposal to introduce a new canonical embedding model (e.g.,
`ants-embed-v2`) must include, at minimum:

- The exact model and weights, identified by BLAKE3 hash per
  [RFC-0008](./RFC-0008-wire-formats.md) §5.
- A perplexity comparison against the current canonical model on a
  public benchmark set covering at least ten languages and eight
  domains.
- A vulnerability analysis: known embedding-poisoning vectors,
  robustness against adversarial inputs, and a comparison to the
  current model's published weaknesses.
- A licensing analysis: open-weight license confirmation, transfer
  rights, any encumbrances.
- An estimate of compute cost for cross-version re-embedding of
  existing cache entries.

A proposal that omits any of these is not ready for review.

### Review

The proposal undergoes a public comment period of **at least eight
weeks** — substantially longer than the one-to-three weeks of ordinary
RFC amendments. During the comment period:

- Any peer may file objections, counter-evidence, or proposed
  alternatives.
- Subsystem maintainers for verification, identity, and cache layers
  each provide a written assessment.
- Independent benchmarks against the current model are encouraged from
  any contributor.

The eight-week minimum is not negotiable downward. It can be extended
if the proposal is contested or if additional evidence is requested by
maintainers.

### Decision

- **During the BDFL period (v0.x)**, the BDFL makes the final merge
  decision after the comment period. The decision is recorded with
  explicit reasoning in [`spec/CHANGELOG.md`](./CHANGELOG.md).
- **After v1.0**, the Technical Steering Committee votes; a **two-thirds
  majority** is required. This is higher than the standard
  simple-majority threshold for routine matters in
  [GOVERNANCE.md](../GOVERNANCE.md), reflecting the consequential
  nature of this decision.

### Transition

Once a new model is accepted:

- A coordinated transition period of **at least six and at most twelve
  months** begins. During this period, peers serve both `v1` and `v2`
  embeddings in parallel, and the DHT routes lookups in both spaces.
- Background workers (incentivised by NCS) re-embed existing cache
  entries from v1 to v2. The producer of the original entry receives
  no additional royalty from the re-embedding (the original work was
  already paid).
- The old model is deprecated when v2 adoption exceeds 95% by traffic,
  OR when the comment period plus transition period reaches twelve
  months, whichever comes first.

### Right to fork

Throughout the proposal, comment, decision, and transition periods,
**any peer or coalition may fork the protocol**. The forked protocol
may adopt a different canonical embedding model or remain on the old
one. The protocol's permissionless property applies here as
everywhere: dissent in code is always a legitimate response.

### What this does not eliminate

The canonical embedding model remains a single point of governance.
The above process **bounds the abuse** of that authority but does not
**dissolve** it. The honest framing: this is one of the few places in
ANTS where the protocol genuinely depends on a coordinated choice, and
the governance process is the only defence against that choice being
made badly. The cost is the eight-week minimum review and the
two-thirds post-v1.0 threshold; the value is that the choice is made
in public, under scrutiny, with the right to dissent always available.

---

## DHT routing — finding the right shard

The cache is sharded across peers by **embedding region**. A region is a partition of the 1024-dimensional embedding space, constructed via locality-sensitive hashing so that semantically similar embeddings tend to land in the same or adjacent regions.

Concretely, the protocol uses a tree-projection LSH scheme: each embedding is reduced to a 64-bit *shard key* by projecting it onto 64 pseudorandom hyperplanes (the projection matrix is part of the protocol spec, fixed once per embedding model version) and taking the sign of each projection. The shard key is the bitmask of those signs.

Two embeddings whose cosine similarity is high will, with high probability, share most of the bits in their shard key. A lookup query computes its own shard key, then queries the DHT for peers responsible for that key — and for nearby keys (Hamming distance 1, 2, sometimes 3) to handle the borderline cases where similar embeddings land in adjacent shards.

The DHT itself is a Kademlia variant (the BitTorrent mainline DHT family), adapted to use shard-keys as node identifiers instead of arbitrary hashes. This piggybacks on a decade of empirical robustness data from the BitTorrent ecosystem.

A peer can choose to host as much or as little of the embedding space as it has storage for. A laptop with 200 GB of free space might host 0.001% of the cache; a NAS with 50 TB might host 5%. Hosting earns NCS continuously, weighted by region popularity and uptime.

---

## The lookup protocol

A consumer query proceeds as follows:

1. The consumer's client computes the embedding of the prompt locally. (CPU work — 50-200ms on commodity hardware.)
2. The client computes the shard key from the embedding.
3. The client queries the DHT for peers responsible for that shard key, plus near-neighbour shards.
4. The DHT returns a small set of candidate peers (typically 5-15).
5. The client sends a parallel `cache_lookup` request to all of them with the embedding and a similarity threshold (e.g., 0.92).
6. Each peer searches its shard for entries whose embedding has cosine similarity ≥ threshold and returns the top-k results (with their metadata).
7. The client aggregates results from all responding peers, deduplicates by prompt-hash, ranks by a composite score (similarity × producer-reputation × positive-signals × recency-factor), and either:
   - **Serves the top candidate** (if its composite score exceeds a threshold the user has set);
   - **Requests fresh inference** (and contributes the new result back to the cache);
   - **Asks for verification** (Tier 2 or Tier 3 from RFC-0003) on the cached candidate before serving.

The user's client chooses among these three based on their declared trust tolerance for the query. A casual conversational query trusts the cache. A query labelled "medical: dosing question" demands fresh inference plus Tier 3 verification regardless of cache quality.

End-to-end latency for cache hits: typically 100-300 ms in good network conditions, dominated by network round-trips, not by computation.

---

## The write protocol

When a fresh inference is produced (because no cache hit was satisfactory), the result is offered for caching.

The producer peer constructs a signed cache-entry record (schema above) and writes it to the DHT. Specifically:

1. The producer computes the shard key from the embedding.
2. It identifies the canonical peer or peers responsible for that shard.
3. It transmits the cache-entry record with its hardware attestation of the inference work.
4. The receiving shard-holders validate the attestation (proving the producer ran what it claims), check that the prompt-hash doesn't already exist (avoiding exact duplicates), and store the entry.
5. The entry is replicated to N (typically 3-5) shard-holders for durability.

The protocol does not require consent from anyone to cache. Caching is a side effect of serving. A peer that performs an inference and refuses to contribute the result to the cache *can* — but they lose the cache-hit revenue stream that would otherwise accrue to them, so the incentive runs strongly against it.

---

## Quality, decay, and invalidation

A cache that grows forever without quality management becomes a cesspool. ANTS specifies three mechanisms that keep it useful.

**Implicit decay by validity class.** Each entry declares its expected useful lifetime (`ephemeral`, `weeks`, `months`, `years`, `perennial`). Beyond that horizon, the entry is downweighted in lookup ranking (it can still be served, but is no longer preferred). After 4× the declared horizon, the entry is eligible for eviction by shard-holders to make room for newer content.

**Explicit invalidation by challenge.** Any peer may challenge a cache entry by re-running the inference (with TEE attestation) and submitting a result. If the new result substantively differs from the cached one — measured by embedding distance of the responses, plus optional human-review escalation — a challenge event is appended. After K independent challenges (currently 3), the original entry is invalidated. The original producer's reputation is downweighted.

**Quality signal aggregation.** Positive and negative human feedback is aggregated per entry. An entry with consistently positive signals climbs the ranking; one with mounting negative signals descends and eventually is hidden from default lookups (though still retrievable for audit). Crucially, this is *aggregated* feedback — the protocol does not let any single peer veto an entry. K independent signed downvotes from K different attested peers are required to trigger demotion.

Combined, these three mechanisms produce a cache that is self-curating without central editorial control. Bad entries decay or get invalidated. Good entries earn their producers rent for as long as they remain accurate.

---

## Adversarial considerations

The cache is a target. The protocol's resistance is built into its primitives, but the threat model deserves explicit enumeration.

**Cache poisoning.** An adversary creates many attested peers and floods the cache with low-quality or actively misleading answers, hoping users serve them. Defences: (a) every entry is signed; the producer's reputation matters at retrieval ranking. (b) Hardware attestation means each fake peer costs real money. (c) Challenge mechanism allows any honest peer to invalidate poisoned entries. (d) Human feedback aggregation surfaces misleading entries quickly. (e) Producer reputation, downweighted by invalidations, causes repeat offenders to lose ability to write to the cache at all.

**Embedding-region squatting.** An adversary tries to dominate a particular embedding region (e.g., medical advice) by writing many entries with cached responses tuned to specific intent. Defence: shard-holders are themselves attested and rotated; a single adversary cannot hold a region exclusively. Lookup queries hit multiple shard-holders in parallel and aggregate, so even if one shard-holder is malicious, honest neighbours provide alternative results.

**Lookup correlation attacks.** An adversary observes many lookups and tries to correlate them with user identities. Defence: lookups are routed through K-hop circuits (similar to Tor) at the user's option; for privacy-sensitive queries, the user pays a small NCS premium for circuit routing.

**DHT eclipse attacks.** An adversary surrounds a target peer in the DHT and feeds it manipulated routing information. Defence: DHT clients use multiple bootstrap paths and consistency-check their views against known-good peers periodically.

**Storage exhaustion attacks.** An adversary writes massive entries to fill the cache. Defence: entry size limits, written-by-peer rate limits scaled to attested hardware (small devices can write less), and per-shard storage quotas with eviction policies that downweight low-popularity entries.

We do not claim the cache is unbreakable. We claim it is *expensive to break* and *self-healing after each break*. This is the same threat model as the rest of the protocol.

---

## Privacy considerations

The cache makes two specific privacy trade-offs explicit.

**Embeddings leak partial information about prompts.** A sufficiently capable adversary, given an embedding and access to the embedding model, can reconstruct an approximation of the original prompt. This is *not perfect inversion* (it loses fine detail) but it is more than zero. Two implications follow:

1. The embedding alone, stored in the cache, is not enough to leak a specific user's prompt — because the cache doesn't know which user wrote it. The embedding lives in a shared semantic space; many users may produce similar embeddings.
2. A peer who observes a lookup query can learn approximately what the user is asking. Mitigation: the K-hop circuit routing option for privacy-sensitive lookups, plus optional embedding perturbation (small random noise added to the query embedding before transmission, trading minor recall for partial obfuscation).

**Cache-hit responses are not private to the requester.** When a cached answer is returned, the serving peer learns *that this question was asked*, but not (under default routing) *who asked it*. Multi-hop routing breaks even this minimal linkage.

For users requiring stronger privacy — medical, legal, financial — the protocol's [Privacy Modes](./RFC-0006-payment-terms.md) specify three tiers: open (default), jurisdiction-bound (peer matched by attested location), and encrypted-TEE-to-TEE (the cache is *not* used in this mode; queries bypass cache to a directly-attested confidential inference peer, at much higher cost).

---

## Integration with the community economy

The cache is the mechanism that resolves the structural pathology of pure barter. The economic interface is straightforward:

- **Cache hosting** earns NCS continuously, based on storage capacity × uptime × shard popularity, paid to the host by the network through a small fraction of every cache-hit revenue.
- **Cache hit** is a service the host provides to the requester. It costs the requester a small fraction (5-15%, network parameter) of the equivalent fresh-inference cost. The host receives this payment in NCS via the standard local-ledger mechanism.
- **Producer royalty.** Of every cache-hit fee, a fraction (currently 50%) flows back to the *original producer* of the cached entry, recognised by their signature on the entry. This is the rent specialists earn from canonical answers. The other 50% goes to the host who served the lookup.
- **Cache-write incentive.** When a peer performs fresh inference, they earn the immediate fee from the consumer *and* the right to royalty on all future retrievals of the cached entry. This is what makes writing to the cache rational rather than hoarding the answer.

Note that in commercial economies (RFC-0006), the same mechanism operates with fiat instead of NCS, and the cache hosts may set their own retrieval prices declared via Payment Terms.

---

## What we have not figured out yet

The places we would most like sharp eyes:

- **Similarity threshold for cache hits.** 0.92 cosine is a guess. Below it, the cache misses too often; above it, we serve answers that aren't actually relevant. Domain-dependent. Probably needs to be configurable per-query.
- **Cache-hit payment fraction.** 5-15% of fresh inference cost is a guess. Field data needed.
- **Producer royalty fraction.** 50% of cache-hit revenue to the original producer is a guess that balances "specialists keep contributing" with "hosts get paid enough to bother."
- **Shard-key bit length.** 64 bits is enough for ~10^14 distinct shards. Probably overkill; might shrink to 48.
- **Replication factor.** 3-5 copies of each entry — is this enough for durability across peer churn? Empirical question.
- **Validity classes.** Should there be more granularity? Fewer? Should producers declare them or should the network infer them from content?
- **Cross-language cache hits.** A question asked in Italian may have its best answer in English. Should the cache cross-route via translation, or keep each language semantically distinct?
- **Cache-write spam from large producers.** A peer with enormous resources could flood the cache with mediocre entries hoping for retrieval royalties. The challenge mechanism handles obvious cases, but the cost/benefit of mediocre-but-not-wrong entries needs thought.

---

## A reference sketch you could implement

A working cache layer is more complex than the barter economy — but the core is still small.

```python
# Heavily simplified. Skips DHT routing, attestation, churn handling, etc.
import faiss
import time
from dataclasses import dataclass

@dataclass
class CacheEntry:
    embedding: bytes        # f32[1024]
    prompt_hash: bytes      # 32 bytes
    response: bytes
    producer: bytes         # peer ID
    attestation: bytes
    created: float
    validity_class: str
    positive: int = 0
    negative: int = 0

class CacheShard:
    """One peer's local view of a region of the cache."""
    def __init__(self, dim=1024):
        self.index = faiss.IndexFlatIP(dim)   # inner product on normalized vectors == cosine
        self.entries: list[CacheEntry] = []

    def add(self, entry: CacheEntry):
        # Verify attestation, then add
        self.entries.append(entry)
        self.index.add(entry.embedding_array().reshape(1, -1))

    def lookup(self, query_embedding, threshold=0.92, top_k=5):
        D, I = self.index.search(query_embedding.reshape(1, -1), top_k)
        results = []
        for sim, idx in zip(D[0], I[0]):
            if sim >= threshold and idx < len(self.entries):
                e = self.entries[idx]
                if self._still_valid(e):
                    results.append((sim, e))
        return sorted(results, key=lambda r: self._score(*r), reverse=True)

    def _still_valid(self, entry):
        age = time.time() - entry.created
        horizons = {"ephemeral": 300, "weeks": 1.2e6, "months": 7e6,
                    "years": 6e7, "perennial": float('inf')}
        return age < 4 * horizons.get(entry.validity_class, 0)

    def _score(self, sim, entry):
        signal_ratio = (entry.positive + 1) / (entry.negative + 1)
        recency = max(0.1, 1.0 - (time.time() - entry.created) / 3.15e7)
        return sim * signal_ratio * recency
```

Around 40 lines of Python for the lookup heart. FAISS handles the vector search; the protocol around it handles DHT routing, attestation validation, replication, and economic accounting. Production-ready it isn't, but it conveys that the core is implementable by one engineer in one weekend.

---

## What comes next

This RFC defines the memory layer. It depends on no other RFC (it can be specified and partially implemented standalone), but it is *depended on by* all of:

- RFC-0001 (community economy) — uses cache for specialist credit
- RFC-0003 (verification) — verifies cached entries in addition to fresh inference
- RFC-0006 (payment terms) — declares how cache retrieval is priced across economies

The next RFC to draft is RFC-0003 (verification), because verification is what allows the cache to maintain quality without central editors.

---

## How to contribute to this document

Open an issue on `Ants-Community/ants` with the tag `rfc-0002`. Particularly valuable contributions: experimental results on similarity-threshold tuning, alternative embedding-model proposals with benchmark data, formal analysis of the LSH partition scheme, prior art we have missed.

The document is CC0.

---

*Drafted by the ANTS founding contributors, May 2026.
Memory is what turns a network of strangers into a colony.*
