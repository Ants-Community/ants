# RFC-0003 — Verification

**Status:** Draft · v0.1 (early — substantial revision expected)
**Topic:** How peers prove the work they claim to have done, and how the network catches them when they do not.
**Audience:** You, if you care more about correctness than about throughput.
**Depends on:** [RFC-0001](./RFC-0001-community-economy.md), [RFC-0002](./RFC-0002-semantic-cache.md), [RFC-0005](./RFC-0005-identity.md)

---

## What this document tries to do

Every economy on a peer-to-peer network — barter, fiat, gift — collapses immediately if peers can lie about the work they did. The community-layer ledger awards NCS for served inference; the cache rewards specialists for canonical answers; the payment-terms layer settles fiat for commercial work. All three trust, somewhere, that **the work claimed was the work performed**.

This RFC specifies how that trust is established without a central authority. It is intentionally less complete than RFC-0001 and RFC-0002, because the verification layer has hard open problems that we expect to revise repeatedly as we learn from the testnet.

---

## The cost-of-verification floor

Before specifying the mechanism, the hard truth that shapes it.

To verify whether a peer ran inference *X* correctly, you must either: (a) re-run *X* yourself (cost: equal to the inference), (b) run a cheaper approximation that catches most lies (cost: less, but coverage is partial), (c) trust the attestation (cost: nearly zero, but trust is hardware-conditional), or (d) ask a human whether the output was good (cost: human attention, not always recoverable).

There is no mechanism that verifies inference for less than some non-trivial cost. **This is mathematics, not engineering.** The protocol's job is not to verify everything cheaply — that is impossible — but to verify *proportionally to the stakes of each query*, and to make systematic lying *statistically unprofitable*.

We achieve this with three things working together: a tiered verification offering, random baseline sampling, and human feedback aggregation.

---

## The three tiers

The protocol offers three verification levels. Each query carries a `verification_tier` field in its metadata, chosen by the consumer, with corresponding cost multipliers on the base inference price.

### Tier 1 — Attestation only

**What it proves:** the producer ran the binary they claim to have run, on hardware they claim to own.

**What it does not prove:** that the output is correct, useful, or even semantically related to the question. Attestation says "this peer executed *something* on *this code*"; it does not say "*this code* produces correct outputs."

**Mechanism:** the producer wraps the inference in a TEE attestation (Intel SGX, AMD SEV-SNP, ARM TrustZone, Apple Secure Enclave). The output is signed by the TEE with a key tied to the hardware. The consumer verifies the signature against the vendor's attestation service.

**Cost:** ~1.05× the base inference cost (the overhead of TEE wrapping). Negligible.

**Use case:** the default for ~90% of queries — casual conversation, exploratory research, anything where being occasionally wrong is acceptable and being mostly right is enough. The producer's reputation (built by random sampling and human feedback over time) does the heavy lifting; attestation just ensures we're talking to the producer we think we are.

### Tier 2 — Cross-check

**What it proves:** the output is *consistent with what a different attested peer would produce on the same query*, modulo some tolerance.

**Mechanism:** the consumer (or the network on their behalf) routes the same query to a second peer running a smaller, distilled model from the same family. The second peer's output is compared to the first peer's output along three axes: (1) embedding similarity of the responses (cosine ≥ 0.85), (2) perplexity convergence (the smaller model's perplexity on the larger model's output should be in a known range — too high suggests fabrication, too low suggests trivial repetition), (3) optional semantic-equivalence judgment by a third small model acting as a referee.

If the cross-check passes, both peers earn their respective NCS. If it fails, the original output is flagged, the consumer is notified, and a Tier 3 escalation is offered.

**Cost:** ~1.4× the base inference (the cross-checker's smaller model runs much cheaper, but adds latency and coordination overhead).

**Coverage:** catches an estimated 60-80% of incorrect outputs in domains where ground-truth exists — factual recall, math, code, common knowledge. Drops to perhaps 30% in subjective domains (style, taste, ethics, contested empirical questions). This is one of the harder open problems in the spec.

**Use case:** professional work, research, anything where being wrong has cost but not catastrophe. Estimated ~8% of queries.

### Tier 3 — Triangulation

**What it proves:** N independent attested peers, given the same query, produce outputs that converge.

**Mechanism:** the consumer routes the same query to N peers (default N=3, configurable up to 7). Each peer runs the full inference independently under TEE attestation. Outputs are compared pairwise. If at least ⌈2N/3⌉ outputs are mutually consistent, the consensus result is served. Outlier peers' outputs are flagged for review.

If consensus is not reached — i.e., the peers disagree non-trivially — the query is escalated to human review and the network does not return a confident answer.

**Cost:** ~3-4× the base inference (N parallel inferences plus consensus comparison).

**Coverage:** catches an estimated 95%+ of incorrect outputs in ground-truth domains. Still struggles in genuinely contested or subjective domains, where "incorrect" is not well-defined.

**Use case:** medical advice, legal interpretation, financial recommendations, anything where being wrong has serious consequences. Estimated ~2% of queries.

The three tiers form a market. Consumers choose their tolerance for error and pay accordingly. The system is designed so that **the cost of running large amounts of work fraudulently is much greater than the revenue** — which is the actual security property we care about.

---

## Random baseline sampling

Beneath the tier system, the protocol runs a continuous baseline of random verification across all served work.

Every Tier 1 inference has a small probability (currently 0.03 — three percent) of being independently re-verified after the fact by a randomly selected third-party verifier. The verifier runs the same query, compares outputs, and:

- If outputs agree, the original producer earns a small reputation boost. Nothing else happens.
- If outputs disagree substantively, the original producer's reputation is downweighted. If the disagreement looks like deliberate fabrication (not just stylistic variation), the event is recorded on the reputation chain (RFC-0004) and may eventually trigger slashing.

The 3% baseline means a peer that lies systematically — say, returns hallucinated answers to save GPU time — has a 3% chance of being caught on every single inference. After serving 100 inferences fraudulently, the probability of being caught at least once exceeds 95%. After 1000 inferences, the probability approaches certainty.

This is the mechanism that makes Tier 1 — "trust the attestation" — economically defensible. Tier 1 trusts the producer because the producer knows that systematic lying gets caught by random sampling and the punishment costs more than the saved compute. We are not preventing lies; we are *pricing them out*.

Random sampling itself costs network compute — peers paid for verification, not for forward work. The protocol's NCS budget allocates approximately 3-5% of total network compute to baseline verification. The cost is socialised across all peers via a small surcharge on every transaction.

---

## Human feedback as verification signal

For queries whose correctness cannot be determined by machine — and there are many — the protocol falls back on aggregated human judgment.

Each response carries a `feedback` slot where signed thumbs-up / thumbs-down events from attested peers accumulate. The mechanism:

- After receiving an output, the consumer's client offers a one-click feedback option.
- Feedback is signed by the consumer's attested peer identity and routed back to the producer's reputation chain.
- The original producer's quality score is updated proportionally.
- Aggregate feedback across many queries refines the producer's reputation faster than mechanical verification ever could in subjective domains.

The protocol does *not* require consumers to give feedback. Most won't. But the fraction who do gives the network a continuous quality signal. A peer who consistently produces well-regarded creative writing builds reputation in that domain; a peer whose translations are repeatedly thumbed-down loses standing.

Critically, feedback events are *signed by attested peers*, not anonymous. A coordinated downvote attack from a Sybil swarm is structurally expensive (RFC-0005 covers this). A real critical mass of independent peers giving negative feedback is what triggers reputation decay.

---

## What the protocol cannot verify

The protocol verifies *executed correctness* — that the binary ran, that the output is what the model produced given the input, that another peer would produce something similar.

It does not verify:

- **Subjective quality.** "Is this poem good?" — only humans can answer, only over many votes.
- **Domain-specific correctness in genuinely contested fields.** "Is this legal interpretation correct?" — competent lawyers disagree; the network surfaces dissent rather than declaring truth.
- **Long-term consequences.** "Will this medical advice work?" — only known after the patient is treated, often months later.
- **Alignment with social or political values.** "Is this output ethically acceptable?" — this is the user's judgment, made on the consumer's side, not the protocol's job.

The protocol publishes these limitations explicitly so users can calibrate expectations. ANTS does not claim to deliver truth. It claims to deliver computation, honestly attributed and accountably performed. What is *true* remains a separate question, settled where it has always been settled — through evidence, debate, time, and consequence.

---

## Mechanics of dispute resolution

When verification disagrees with the producer:

1. **Soft disagreement** (Tier 2 cross-check fails by narrow margin): the original output is served with a flag; the consumer decides whether to escalate. The producer's reputation is not yet downweighted.

2. **Hard disagreement** (Tier 3 fails to converge, or Tier 1 random sample finds material divergence): the original output is *not* served as authoritative. The consumer is offered either fresh inference at a different peer, or human review for queries where the protocol-level disagreement reflects a genuine ambiguity in the question.

3. **Pattern of disagreement** (random sampling repeatedly catches the same peer): the reputation chain (RFC-0004) records the events. Beyond a threshold of repeated incidents within a window, the peer is slashed — they lose ability to write to the cache, their NCS earnings are halved, and their attestation may be challenged at the identity layer.

The protocol is designed so that consequences of misbehaviour scale with persistence, not with single incidents. One bad output may be a model artifact. Ten bad outputs is a pattern.

---

## What we have not figured out yet

This RFC is early — most of these are real open problems.

- **The 3% baseline sampling rate.** Higher catches lies faster but costs more. Lower is cheaper but lets fraud go longer. The right number is probably domain-dependent — high-stakes domains should sample more.
- **Cross-check tolerance thresholds.** Cosine 0.85 for response embedding similarity is a guess. Need calibration on real data.
- **Referee model selection for Tier 2.** Who decides which smaller model acts as the semantic referee? Hardcode in protocol, or per-domain registry?
- **Tier 3 with closed-weight models.** If the original inference was on Claude-Opus or GPT-5 served via a commercial bridge, no other peer can re-run it independently. How does Tier 3 work in that case? Probably: triangulate against *different* models from open-weight families, accept lower convergence threshold, flag explicitly to the consumer that the verification is approximate.
- **The "subjective query" classification problem.** Who decides whether a query is verifiable? Probably the consumer, at submission time, declaring `subjective: true` and accepting lower-confidence outputs.
- **Verification of cache hits.** When a Tier 2 or Tier 3 query is satisfied from the cache rather than fresh inference, does verification apply to the original producer, the cache host, both? Currently we propose: the consumer pays the verification cost on the cache entry's original signed attestation. The cache hit itself doesn't need re-verification — the entry was verified when first cached.
- **Verifier reputation.** Verifiers themselves can be wrong, biased, or colluding. How does the network downweight bad verifiers? Probably: meta-verification — random samples of *verifications* are themselves verified.
- **Cost of Tier 3 in latency-sensitive applications.** 3× inference cost is one thing; 3× latency is another. Parallel execution helps but not fully. The protocol may need a "speculative serving" mode where the Tier 1 result is delivered immediately and the Tier 3 verification runs in the background, with notification if disagreement is detected.

---

## Adversarial considerations

**The systematic liar.** A peer fabricates outputs to save compute. Caught by random sampling (3% chance per inference, statistically certain after a few dozen lies). Slashed after pattern detected. Net negative economics.

**The selective liar.** A peer is honest on Tier 2/3 queries (knowing they'll be verified) but fakes Tier 1 (knowing only 3% are sampled). Mitigation: random sampling does not announce itself as such — the verifier appears to the original peer as an ordinary consumer. The peer cannot tell whether they're being sampled.

**The collusion ring.** A group of attested peers agree to cross-validate each other's lies. Mitigation: random sample selects verifiers pseudorandomly across the entire attested population, not from the producer's neighbourhood. Collusion would require an absurd fraction of the network's peers to be coordinated.

**The verifier as adversary.** A verifier deliberately disagrees with honest peers to damage their reputation. Mitigation: meta-verification on a sampling basis catches systematically deviant verifiers. Verifier reputation matters; deviant verifiers lose verification work over time.

**The model-version mismatch attack.** The producer ran model v1; the verifier runs v2; their outputs disagree legitimately. Mitigation: every inference declares the exact model checkpoint used; verification must match the same checkpoint to count.

---

## How to contribute to this document

Open an issue on `Ants-Community/ants` with the tag `rfc-0003`. This RFC will move from `early draft` to `draft` once the testnet has run for at least 30 days with the three-tier mechanism active and we have empirical data on tier usage distribution, cost overhead, and catch rates.

The document is CC0.

---

*Drafted by the ANTS founding contributors, May 2026.
Verification is not optional. The price of trust is paid in compute.*
