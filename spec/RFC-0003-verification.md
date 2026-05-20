# RFC-0003 — Verification

**Status:** Draft · v0.2
**Topic:** How peers prove the work they claim to have done — at the right cost for the stakes of each query, on a mechanism the test can actually rest on.
**Audience:** You, if you care more about correctness than about throughput.
**Depends on:** [RFC-0001](./RFC-0001-community-economy.md), [RFC-0002](./RFC-0002-semantic-cache.md), [RFC-0004](./RFC-0004-reputation-pouh.md), [RFC-0005](./RFC-0005-identity.md)

---

## Another Tuesday evening

It is still Tuesday evening. The peer in Trento has answered your question about
Italian industrial-construction law. The answer came back in 800 milliseconds.

You did not pay for it because you were already paying. That is RFC-0001.
The answer was reused from the cache, signed by its producer eight months ago,
retrievable by similarity. That is RFC-0002. But neither of those documents
answers the question this one is for:

**How do you know the peer in Trento didn't make the answer up?**

It could have run a model a tenth the size and charged you for the large one. It
could have run the right model at half the precision to save power. It could have
returned a cached answer to someone else's question. You have no way, locally, to
tell a brilliant answer from a plausible lie — that is precisely why you asked
someone else.

The economic layer assumed this problem was solved. It is not. This document is
about how close we can get, and — in the project's habit — about exactly where
we cannot get there yet.

---

## What we are trying to prevent

A producer **B** can defraud a requester **A** in six distinct ways. They are
not "right answer / wrong answer." They are:

1. **Model substitution** — run a 7B, bill for a 70B.
2. **Degradation** — run M but quantised, context truncated, fewer steps.
3. **Lazy fraud** — return noise, or a cached answer to a different question.
4. **Backdoor** — run M′ with poisoned or censored weights.
5. **Replay** — return a valid answer computed for someone else.
6. **Infidelity** — correct weights, dishonest runtime.

The requirement: **A** gains confidence that `y = M(x)` *without re-running M(x)*
— because if A could cheaply re-run it, A would not have asked B. This is
verifiable computation, specialised to LLM inference, in a permissionless and
economically adversarial network. It is one of the two genuinely hard layers of
ANTS. The other is [RFC-0004](./RFC-0004-reputation-pouh.md). They are
connected; we will show where.

---

## The cost-of-verification floor

To verify whether a peer ran inference *X* correctly, you must either: (a) re-run
*X* yourself (cost: equal to the inference); (b) run a cheaper approximation that
catches most lies (cost: less, but coverage is partial); (c) trust hardware
attestation (cost: nearly zero, but conditional on the hardware vendor); or
(d) ask a human (cost: human attention, not always recoverable).

There is **no mechanism that verifies inference for less than some non-trivial
cost**. This is mathematics, not engineering. The protocol's job is not to
verify everything cheaply — impossible — but to **price verification proportionally
to the stakes of each query** and to make systematic lying *statistically
unprofitable*.

We do this with two ideas held together: a **tiered menu** that lets the
requester choose what assurance they pay for, and a **single rigorous primitive**
that all tiers rest on. The previous draft had only the menu; the draft before
that had only the primitive; v0.2 is the marriage of both.

---

## The shape of the answer: a tiered menu, one primitive

The naive instinct is: *prove the answer is honest before sending it.* It is
wrong, and the manifesto-shaped reason matters here. A program running on
hardware the adversary controls cannot, by itself, produce an unforgeable proof
of its own honest execution — a compromised agent can always emit the bytes an
honest proof would emit. Unforgeability must come from outside the operator's
power: a hardware key it cannot extract, a cryptographic proof it cannot fake,
or **a commitment it cannot satisfy unless it actually did the work, checked
later, at random, with a severe penalty**.

The first two re-centralise (TEE) or are unaffordable today (zero-knowledge
proofs of full LLM inference). The third is what scheme (C) is, and it sits
under every tier below. The temporal frame is not *prove-before-send*. It is:

> **commit-at-send · prove-if-challenged · lose-everything-if-caught**

What changes per tier is *who is challenged, how often, and with what
redundancy*. The primitive — what counts as a commit, what counts as a valid
proof, what counts as a slash — is one and the same across the menu.

---

## The unified primitive: commit-at-send

When a producer **B** answers a query for **A**, it does not send only the
answer. It sends a structured commitment to *the whole computation it claims to
have done*:

```
commit = (
    root         = MerkleRoot(per-position-leaves),    # leaf_i = H(dist_i ‖ i)
    H(M)         = hash of the model weights claimed,
    x            = the input,
    req_nonce    = fresh nonce supplied by the requester,
    agent_meas   = measurement of the canonical agent runtime,
    round        = r,
)
```

The Merkle binding is **position-binding**: B is irrevocably committed to the
distribution at every position of the response, before any verifier asks which
positions to open. The `H(M)` ties the commitment to a specific model checkpoint
(model substitution becomes verifiable). The `req_nonce` ties it to this query
(replay dies). The `agent_meas` ties it to a public, attested runtime measurement
(runtime infidelity dies). The whole object is signed by B's attested identity
(forgery dies).

### Anti-grinding: where the challenge comes from

A naive non-interactive design — *let B compute its own challenge from
H(commit)* — is **insecure here**, and seeing why is load-bearing. B controls
part of the preimage and can grind: try variants until the derived challenge
lands only on positions B chose to compute honestly. Fiat–Shamir on the
prover's own commitment fails for small challenge sets like ours.

So the challenge must come from public entropy **posterior to and unbindable
by** B's commitment. RFC-0004's PoUH chain produces exactly such a beacon: each
finalized block emits a fresh, committee-attested random value `B_{r+1}` whose
content B cannot influence (B fixed `root` at round `r`; the beacon arrives
after).

From `B_{r+1}`, three things are derived deterministically — anyone can
recompute them, no verifier has discretion:

```
audit?  =  PRF(B_{r+1} ‖ root ‖ "sel")  <  p · 2^k     (does this commit get checked?)
S       =  Expand( PRF(B_{r+1} ‖ root ‖ "pos") , m , L )   (which positions, if so)
V       =  ReputationSet[ PRF(B_{r+1} ‖ root ‖ "aud") ]    (who audits, k-of-n)
```

The same primitive defeats grinding (B fixed `root` before `B_{r+1}` existed),
prediction (nobody knew the challenge until the beacon arrived), and auditor
collusion (B cannot choose, be, or foreknow its auditor).

### Canonical numerics on the audited path

LLM inference is *not bitwise reproducible* across honest hardware. Different
GPUs, different kernel choices, different FlashAttention versions all produce
slightly different logits for the same input. So "re-execute and check it
matches" can never be exact-match.

The protocol mandates a **canonical numerical recipe** — fixed reduction order,
deterministic kernels, mandated math flags — *for the committed path only*.
B's user-facing answer can be served on whatever kernel is fastest; the trace
committed in the Merkle root must follow the canonical recipe. This is the
"verifiability tax": you cannot use your fastest kernel on the path that gets
audited.

This is the design lever. With canonical numerics, honest cross-hardware
divergence shrinks toward the pure rounding floor; the gap between "honest
hardware diversity" and "dishonest quantization" widens; aggregate tests become
viable.

---

## Tier 1 — attestation plus baseline sampling

**Default for ~90% of queries.** Casual conversation, exploratory research,
anything where being occasionally wrong is acceptable and being mostly right is
enough.

**What it commits to:** the producer wraps the inference in a TEE attestation
(Intel TDX, AMD SEV-SNP, NVIDIA confidential GPU, ARM CCA, Apple Secure Enclave)
in addition to the commit-at-send object above. The TEE signs the
`(root, H(M), x, req_nonce, agent_meas)` with a key tied to attested hardware.

**Cost:** ~1.05× the base inference cost — the wrapping overhead. Negligible.

**What catches systematic fraud:** a small fraction `p` of Tier-1 inferences
are independently audited after the fact. The auditor is beacon-selected, the
challenge positions are beacon-selected, the test is the aggregate one of Tier
2 below. The auditing peer cannot announce itself; B cannot tell whether any
given query is being sampled.

**How `p` is set:** not from intuition. From the economics. A producer that
saves `G` units of compute per fraud and faces a slash penalty `S_NCS` (RFC-0001
defines `S_NCS` as a derived quantity) earns negative expectation iff

```
p  >  p_min  =  G / ( (1 − β) · S_NCS )         (deterrence floor)
```

where `β` is the per-audit miss rate. The cost of running the audits caps `p`
from above:

```
p  ≤  p_max  =  ρ_budget · L / m                (budget ceiling)
```

with `ρ_budget` the verification overhead budget (default 10%), `L` the response
length, `m` the number of positions opened per audit. Tier 1 is viable iff
`p_min ≤ p_max`, i.e., iff there exists a sampling rate that simultaneously
deters fraud and stays within budget.

For the parameter ranges currently estimated for community-economy queries,
`p ≈ 0.03` (the three percent that the v0.1 draft of this document presented as
a guess) sits comfortably inside that band for most fraud classes. The
three-percent number remains as a *default example*; the deriving inequality is
what the reference client actually uses, so the rate adapts as `S_NCS`, `G`, or
`L` shift.

---

## Tier 2 — cross-check via scheme (C)

**For professional work where being wrong has cost but not catastrophe.** ~8%
of queries.

This is the technical heart of the verification layer. It is also where the
v0.1(early) cross-check (compare embeddings, perplexity convergence, cosine ≥
0.85) **fails** for the most dangerous fraud class.

### Why per-position thresholds are not enough

The hard fraud is not "7B pretending to be 70B" — that is structurally easy to
catch, distributions disagree grossly. The hard fraud is **degradation** —
particularly FP8 inference billed as BF16. Modern FP8 is designed to be
near-lossless: the per-position discrepancy it introduces is comparable in
magnitude to the honest cross-GPU noise between two correct implementations.

A per-position rule (similarity ≥ 0.85, or any single-position threshold) has
no choice but to either let FP8 fraud pass (loose threshold) or slash honest
peers on exotic hardware (tight threshold). The per-position ε does not exist
for the FP8 case.

What rescues the scheme is a single asymmetry: **honest numerical noise is
roughly unbiased** (a correct implementation does not systematically push the
distribution in one direction); **quantization is systematically biased**
(degraded models consistently disagree with the reference in the same direction
— they never improve perplexity, they always worsen it). The right object is
not a per-position distance but a **sequential test on the accumulated bias**
over many challenged positions.

### The test: an anytime-valid betting e-process

At each challenged position `j`, the auditor independently recomputes the
reference next-token distribution (with its own copy of `M`, under the canonical
numerical recipe), then forms a **bounded** per-position score
`X_j ∈ [0, 1]` (a clipped distributional discrepancy — total variation or
clipped KL, the exact form is a tuning parameter).

The hypotheses are stated on *conditional means*, not on i.i.d. distributions:

```
H0 (honest):   E[ X_j | F_{j−1} ]  ≤  μ0                 (no independence assumed)
H1 (fraud):    E[ X_j | F_{j−1} ]  ≥  μ0 + Δ
```

`μ0` is the calibrated honest-noise floor under the canonical recipe; it is the
output of the b2 testnet experiment below. The test is constructed by betting.
With a predictable bet `λ_j ∈ [0, 1/(1−μ0))` chosen from data strictly before
position `j`:

```
M_n  =  ∏_{j=1}^{n} ( 1 + λ_j · ( X_j − μ0 ) ) ,    M_0 = 1
```

Under H0, each factor has conditional expectation ≤ 1, so `M_n` is a non-negative
supermartingale starting at 1. By **Ville's inequality**:

```
under H0:   P( ∃ n :  M_n ≥ 1/α )  ≤  α
```

Slash when `M_n ≥ 1/α`. The Type-I error — wrongly slashing an honest peer —
is at most `α` **uniformly over every sample size and every stopping rule**,
with **no independence assumption and no variance assumption**. This matters
more than anything else in this section: the error the network cannot afford,
the one that would betray the manifesto, is the one placed on the most rigorous
footing.

### The cost of catching fraud

Expected detection time under fraud (the analogue of "how many positions must
we open") follows from standard betting-test analysis:

```
E[ m | fraud ]  ≈  ( 2 / e² ) · log(1/α) · τ
                   e = Δ / σ    (systematic bias / honest-noise std — the effect size)
                   τ = integrated autocorrelation time of the score along the trace
```

`τ` is **not fate**: it is set by the challenge-sampling schedule. Contiguous
blocks → large τ. Positions spread/strided across the trace → τ → 1. **Spread
sampling is therefore a derived protocol requirement**, not a tuning
preference: the `S` set drawn from the beacon must be strided, not clustered.

### The economic threshold

Combining the deterrence and budget constraints with the detection-time
expression yields a closed-form viability condition in the primitives of
RFC-0001:

```
                 ┌                                      ┐ ½
   e  ≥  T  =    │  2 · τ · G · log(1/α)                │
                 │ ──────────────────────────────────── │
                 └  ρ_budget · L · S_NCS                ┘
```

Tier 2 is economically defensible for a given fraud class iff `e ≥ T`. When the
effect size measured for that class clears the threshold, a challenge rate
exists that simultaneously deters and stays within budget; when it does not,
that fraud class falls back to Tier 3.

The threshold makes the **canonical-numerics mandate** pivotal. Smaller σ
(tighter recipe) raises `e`. The recipe is the design knob; the verifiability
tax is its cost; the threshold is the test of whether the tax is worth paying
for each class of fraud.

### What the cross-check is, in practice

The cross-checker is *another peer* (beacon-selected from the verifier set)
that runs the reference inference on the opened positions only — one
teacher-forced forward pass per challenged position, not full regeneration. A
modest cross-checker can audit many positions cheaply; the cost is `~ (m/L) ·
C_inf` per audit, not `C_inf`. The v0.1(early) draft's "smaller distilled
model" remains a valid implementation choice for the cross-checker, but the
*test* it runs is the e-process above, not a per-position cosine.

---

## Tier 3 — triangulation under (C)

**For high-stakes work where being wrong has serious consequences.** Medical
advice, legal interpretation, financial recommendations. ~2% of queries.

The consumer routes the same query to `N` peers (default `N = 3`, configurable
up to 7), each running the full inference independently. **Each peer publishes
its own commit-at-send** in the sense of Tier 2 — its own Merkle root, its own
attestation, its own beacon-bound exposure to audit. The per-peer
anti-grinding guarantee of (C) holds for each of the `N` independently.

On top of those `N` commits, a **convergence test** compares the published
outputs pairwise. If at least `⌈2N/3⌉` outputs are mutually consistent
(measured by output-embedding cosine ≥ 0.93 with the consensus median, and
optional semantic-equivalence judgement by a third small model), the consensus
result is served to the consumer. Outliers are flagged. If consensus does not
emerge, no confident answer is returned and the query is escalated to human
review.

**Cost:** ~3-4× the base inference.

**Coverage:** catches an estimated 95%+ of incorrect outputs in domains where a
ground truth exists. Subjective domains continue to need the human-feedback
mechanism below.

The relationship to Tier 2 is: triangulation does not *replace* the e-process;
it *adds* a multi-party convergence test on top of `N` independent (C)
guarantees. A Tier 3 audit can therefore catch a fraud that escapes Tier 2 on
any single peer — through the convergence comparison — without weakening any
single peer's individual verification.

---

## Human feedback as a quality signal

For queries whose correctness cannot be settled by machine — taste, style,
contested empirical questions, subjective interpretation — the protocol falls
back on aggregated human judgement.

A response carries a `feedback` slot where signed thumbs-up / thumbs-down events
from attested peers accumulate. The mechanism is intentionally simple:

- Feedback is **signed by the consumer's attested peer identity** (RFC-0005)
  and routed back to the producer's reputation record. Single-peer feedback
  cannot decisively swing a producer's reputation; what triggers a quality
  signal is **K independent attested signers** within a window (default K = 5).
- The protocol does **not** require feedback. Most consumers won't give it.
  The fraction who do gives the network a continuous quality signal in
  domains where machines cannot.

Feedback events are gossiped, signed, and tied to attested identity — so a
coordinated downvote attack from a Sybil swarm is structurally expensive
(RFC-0005's economics). A real critical mass of independent attested peers
giving negative feedback is what causes a producer's quality multiplier to
decay in subjective domains.

The feedback layer **does not produce slash events**. It only adjusts the
quality multiplier in RFC-0001's choke/unchoke loop and the cache ranking in
RFC-0002. Hard slashing of a peer still requires the cryptographic
verification of one of the tiers.

---

## Cache verification (RFC-0002 integration)

The semantic cache changes a real fraction of "verification" into "verify the
producer-signed entry, not the request you just served." Specifically:

- When a Tier 1 query is satisfied from the cache, no fresh verification is
  needed — the entry was verified at the moment it was first cached, and is
  retrievable with its original attestation. The cache host's job is to serve
  the signed entry, not to re-verify.
- When a Tier 2 or Tier 3 query asks for a verified result and finds a
  cached candidate, the consumer's client can either accept the
  cache-attested verification (cheaper) or commission fresh inference plus
  scheme (C) (more expensive). The choice is the consumer's, declared
  per-query.
- The challenge mechanism of RFC-0002 — any peer may invalidate a cached
  entry by submitting a divergent re-inference with attestation — *is* the
  verification mechanism of Tier 2 applied to cache content. The same
  e-process test, the same canonical numerics, the same threshold.

Verification of cache hits and verification of fresh inference are therefore
the same primitive in two different deployment contexts. The implementation
shares code; the spec shares mechanism; the reasoning shares structure.

---

## Where this bottoms out

The verification layer's guarantees are conditional on three external pieces:

```
RFC-0005 (identity)        →  TEE attestation actually proves unique hardware
RFC-0004 (reputation)      →  beacon is unpredictable & non-grindable;
                              honest majority of the witness layer
RFC-0001 (economy)         →  S_NCS is the slash value (derived in RFC-0001 v0.3)
```

The chain has a fixed point: it terminates at RFC-0004's bounded social
coordination assumption. Verification depends on identity (Sybil resistance);
identity depends on the credible-fork-threat (RFC-0004's social root). This is
exactly the trust-closed argument the corpus has carried since v0.1: the
recursion closes; it does not diverge into hidden centralization.

One residual is worth naming explicitly here, not somewhere else: an adversary
who can adapt the fraud to keep `E[X_j|F_{j−1}] ≤ μ0` *on the positions that
get challenged*, while degrading elsewhere, escapes the statistical test.
This is defeated **only** by (a) the commitment being over all positions
(Merkle binding) and (b) the challenge positions being unpredictable
(beacon-derived). That is a cryptographic requirement, not a statistical one,
and it is precisely what the unified primitive provides. The honest seam
between statistics and cryptography is here.

---

## What we have not figured out yet

The places we would most like sharp eyes:

- **The effect size `e` is unmeasured.** The whole viability call is `e ≥ T`,
  and `e` for the FP8 case under a real canonical runtime is not known. The
  empirical literature suggests it sits near the threshold, which is the
  uncomfortable place. The b2 experiment below measures it.
- **The honest-noise null calibration `μ0`** must be measured on real
  heterogeneous hardware before the deterrent is real. b2 again.
- **The patient adversary with a one-shot decisive attack.** A producer
  willing to take a single slash to execute one high-payoff fraud is not
  deterred by the sampling-rate inequality; the deterrent is statistical,
  not absolute. This residual is shared with RFC-0004.
- **Subjective-domain detection.** Who decides whether a query is
  verifiable? Probably the consumer at submission time, declaring
  `subjective: true` and accepting lower-confidence outputs. Mechanism not
  yet specified.
- **Closed-weight model verification.** If the original inference was on
  Claude or GPT served via a commercial bridge (RFC-0006), no other peer can
  re-run it independently. Tier 3 in that case triangulates against
  *different* open-weight models with an explicitly looser convergence
  threshold, flagged to the consumer. Specifics pending.
- **Verifier reputation.** The auditors are themselves attested peers with
  reputation; meta-verification (sampling of audits) keeps them honest. The
  full mechanism is sketched but not fully specified.
- **Cost of Tier 3 in latency-sensitive contexts.** 3× compute is one thing;
  3× latency is another. A "speculative serve" mode — Tier 1 result
  delivered immediately, Tier 3 verification in the background with
  notification on disagreement — is on the table but not yet specified.

---

## Adversarial considerations

**The systematic liar.** Fabricates outputs to save compute. Caught by Tier 1
sampling at rate `p`; statistical certainty after a few dozen lies; slashed
once the e-process crosses `1/α`. Net negative economics by construction.

**The selective liar.** Honest on Tier 2/3 (knowing they'll be checked),
fakes Tier 1 (knowing only fraction `p` is sampled). Defence: the audit looks
to B like an ordinary consumer query — B cannot tell whether they are being
sampled. The selective liar is therefore exactly the systematic liar, sampled.

**The collusion ring.** A group of attested peers cross-verify each other's
lies. Defence: the auditor is beacon-selected from the entire attested
population (RFC-0004), not from the producer's neighbourhood. Collusion at
scale requires controlling an absurd fraction of the network.

**The verifier as adversary.** A verifier deliberately disagrees with honest
peers to damage their reputation. Defence: the e-process Type-I bound holds
for *any* selection rule, so a malicious verifier cannot manufacture a
false-positive against an honest B beyond rate `α`; they can only fail to
report a true fraud (false-negative), in which case the opened leaves are
public and the verifier itself is slashable for clearing a provable fraud
(transitive verifiability).

**The grinding adversary.** Computes its own challenge before committing.
Defeated by beacon-binding: B publishes `root` at round `r`; `B_{r+1}` is
released after, unbindable by B. No grinding path exists.

**The model-version mismatch.** B claims model M; the verifier runs M′
(different checkpoint); their outputs disagree legitimately. Defence:
`H(M)` is in the commit, the verifier loads the matching checkpoint, or no
verification happens (and the commit is invalid).

---

## b2 — the pre-registered testnet experiment

This document is specifiable at day zero but **not testable** until a testnet
exists, a canonical runtime is implemented, and multi-vendor hardware is
available. What we commit to now is the *experiment and its decision rule*,
fixed in advance so the threshold cannot be retrofitted to produce a
convenient answer:

```
GATE 1 — SAFETY (must pass first):
  Across ≥3 real backends × ≥2 attention kernels, every honest input-entropy
  regime satisfies the conditional-mean null  E[X_j | F_{j−1}] ≤ μ0  with margin.
  Fails  ⇒  Tier 2 is unsafe in that regime  ⇒  fall back to Tier 3 only.

GATE 2 — VIABILITY (only if 1 passes):
  ∃ a canonical recipe R* with verifiability-tax ≤ budget such that, for every
  adversary strategy with G ≥ deterrence-min,
       LCB( e | R*, strategy, regime )  ≥  UCB( T | RFC-0001 params ).
  True   ⇒  Tier 2 GO in that regime.
  False  ⇒  Tier 3 fallback in that regime.
```

σ is measured from the *worst plausible honest pair*, not the average — an
honest peer on exotic hardware must never be slashed. Δ is measured against
the *adaptive* adversary, including the one that dithers to mimic honest
variance: an adversary forced to dither `Δ → 0` has destroyed its own quality
or spent the compute it tried to save, which is a win, and b2 must quantify
the (Δ, realized-G) trade-off, not a single Δ.

Estimation is by block bootstrap because the data is heavy-tailed and
correlated; decisions are on confidence bounds, never point estimates. The
output is a pre-registered table

```
   (model size  ×  input-entropy regime)  →  { Tier 2 GO via R*  |  Tier 3 fallback }
```

published in this RFC at v0.3, after the testnet has run.

A per-regime split is an *expected and acceptable* outcome, not a failure:
Tier 2 may be GO for factual queries and fallback-to-Tier-3 for high-entropy
ones. An honest, actionable result beats a uniform claim we cannot defend.

---

## A reference sketch

In the habit of the other RFCs — small enough to argue with, not
production-grade:

```python
def audit(answer, commit, beacon_next, M_ref, alpha=1e-6, mu0=MU0, cap=CAP):
    # Public, deterministic — anyone can recompute. No verifier discretion.
    if prf(beacon_next, commit.root, b"sel") >= commit.p * (1 << 64):
        return "not-audited"

    S = strided(expand(prf(beacon_next, commit.root, b"pos"), commit.m, commit.L))

    capital, mu_hat, var_hat, n = 1.0, mu0, 1.0, 0
    for i in S:
        dist_i, path_i = answer.open(i)
        assert merkle_ok(path_i, dist_i, commit.root, i)

        p_i = M_ref.reference_distribution(answer.prefix(i))   # canonical numerics
        x   = min(discrepancy(p_i, dist_i), 1.0)               # bounded score

        lam = clip((mu_hat - mu0) / var_hat, 0.0, cap)         # predictable bet
        capital *= 1.0 + lam * (x - mu0)
        if capital >= 1.0 / alpha:
            return "FRAUD"                # Ville: P(this | honest) ≤ alpha

        n += 1
        mu_hat  += (x - mu_hat) / n
        var_hat += ((x - mu_hat) ** 2 - var_hat) / n

    return "passed"   # not "honest" — only: not caught this time
```

The last comment is the whole philosophy. The network never proves a peer
honest. It makes dishonesty unbluffable and unprofitable, and it remembers.

---

## How to contribute to this document

Open an issue on `Ants-Community/ants` with the tag `rfc-0003`. Same three
paths as the other RFCs: clarity PRs (merged quickly); design-question issues
(discussed in the issue, then resolved with a PR or escalated to Matrix);
amendments drafted as patches with rationale. Decisions land in
`spec/CHANGELOG.md`.

The two claims we will most defend, and most want broken: that Type-I error
is rigorously controlled by the betting e-process *without independence or
variance assumptions*, and that the v0.2 marriage of tier framing and unified
primitive composes correctly — that the tier menu does not depend on
anything `commit-at-send` does not provide, and that `commit-at-send` does
not depend on anything outside RFC-0004's beacon and RFC-0005's identity.

The document is CC0. Quote it. Disagree with it in public. Fork it.

---

*Drafted by the ANTS founding contributors, May 2026.
Verification is not optional. The price of trust is paid in compute.*
