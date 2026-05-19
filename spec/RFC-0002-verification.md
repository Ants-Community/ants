# RFC-0002 — Verification

**Status:** Draft · v0.1
**Topic:** The verification layer of ANTS — how a peer trusts an answer it did not compute.
**Audience:** You, if you are willing to disagree in writing.

---

## Back to the Tuesday evening

In [RFC-0001](./RFC-0001-barter.md) you asked your laptop a question about Italian
industrial-construction law, and 800 milliseconds later an answer came back from a
peer in Trento who had spent two years tuning a model on exactly that domain.

You did not pay for it. RFC-0001 explained why.

This document answers a different question, and it is the harder one: **how did you
know the peer in Trento did not lie to you?**

It could have run a model a tenth the size and charged you for the large one. It
could have run the right model at half the precision to save power. It could have
served you a cached answer to someone else's question. It could have run a copy of
the model with a backdoor that changes exactly one kind of answer. You have no way,
locally, to tell a brilliant answer from a plausible lie — that is precisely why you
asked someone else.

The economic layer assumed this problem was solved. It is not. This document is
about how close we can get, and — in the project's habit — about exactly where we
cannot get there yet.

---

## What we are trying to prevent

When peer **A** asks peer **B** to run model **M** on input **x**, **B** can defraud
**A** in six distinct ways. They are not "right answer / wrong answer." They are:

1. **Model substitution** — run a 7B, bill for a 70B.
2. **Degradation** — run M but quantised, context truncated, fewer steps.
3. **Lazy fraud** — return noise, or a cached answer to a different question.
4. **Backdoor** — run M′ with poisoned or censored weights.
5. **Replay** — return a valid answer computed for someone else.
6. **Infidelity** — correct weights, dishonest runtime.

The requirement: **A** gains confidence that `y = M(x)` *without re-running `M(x)`* —
because if A could cheaply re-run it, A would not have asked B. This is verifiable
computation, specialised to LLM inference, in a permissionless and economically
adversarial network. It is one of the two genuinely hard layers of ANTS. (The other
is [RFC-0004](./RFC-0004-identity.md). They are connected, and we will show exactly
where.)

---

## Why there is no single answer

It is tempting to want one clean mechanism. There is not one. As of 2026, **no
mechanism is simultaneously trustless, cheap, hardware-neutral, and Sybil-resistant
for frontier-model inference.** Anyone who tells you otherwise is selling something.
The five families, each with the wall it hits:

| Mechanism | What it proves | Why it is not enough alone |
|---|---|---|
| **TEE attestation** | code X ran in an enclave | not "honest inference of M"; GPU-TEE is datacentre-only → re-centralises; inherits vendor breaks; rented cloud TEEs defeat the Sybil case |
| **ZKML** | `y = M(x)`, trustless, no hardware | proving a full LLM forward pass costs 10³–10⁶×; not viable for interactive inference this decade |
| **Optimistic re-execution** | fraud detectable after the fact | verifier needs M and the compute; LLM inference is **not bitwise reproducible** across honest hardware |
| **Statistical (traps, quorum)** | a *probabilistic* bound on a peer's lie-rate | proves no specific answer; quorum costs N× |
| **Economic backstop** | fraud made **−EV**, not impossible | a deterrent, not a guarantee; a residual always remains |

So we do not pick one. We specify a posture in which **the requester chooses the
assurance level and pays for it**, and the cheapest layer runs always.

---

## The shape of the answer

The naive instinct is: *the agent should prove the answer is honest before it sends
it.* This instinct is wrong, and seeing why is the whole design.

A program running on hardware the adversary controls cannot, by itself, produce an
unforgeable proof of its own honest execution — a compromised agent can always emit
exactly the bytes an honest proof would emit. Unforgeability must come from outside
the operator's power: a hardware key it cannot extract (TEE), a proof it cannot fake
(ZKML), or **a commitment it cannot satisfy unless it actually did the work, checked
later, at random, with a severe penalty.** The first two re-centralise or are
unaffordable. The third is implementable in 2026.

So the temporal frame is not *prove-before-send*. It is:

> **commit-at-send · prove-if-challenged · lose-everything-if-caught**

The earliness you want is preserved — not as a proof before sending, but as an
**irrevocable binding at the moment of sending**. The lie is locked in at *t = 0*;
it is merely *exposed* at *t = challenge*. The agent, when it answers, publishes a
binding commitment to the entire computation. With some probability it is later
challenged on a part of it. It cannot know in advance whether, or where. Cheating is
therefore not impossible — it is **unbluffable and not worth it.** This is the
BitTorrent philosophy of RFC-0001 applied to correctness: not "cheating cannot
happen" but "cheating does not pay."

### The tiered posture

- **Tier 0 — always on, ≈ free.** Reputation plus *trap queries*: the network
  continuously injects questions whose correct M-output is known, and statistically
  bounds each peer's lie-rate. Feeds the quality multiplier of RFC-0001. Catches
  gross fraud; bounds subtle fraud probabilistically.
- **Tier 1 — cheap, sampled.** The commit-and-challenge scheme above. The body of
  this document.
- **Tier 2 — costly, opt-in.** Redundant quorum or full re-execution, for
  high-stakes queries the requester elects to pay N× for.
- **Tier 3 — aspirational, no date.** TEE attestation where the hardware allows it;
  ZKML if it ever becomes viable. Named as the asymptotic ideal. **Explicitly not a
  precondition for the network to exist.**

The rest of this document specifies Tier 1, because Tier 1 is where the design is
load-bearing and where it can break.

---

## The hard case: degradation

Substituting a smaller model is easy to catch — two different models disagree
grossly on most non-trivial inputs. The dangerous fraud is **degradation**, and its
sharpest instance is running FP8 while billing FP16/BF16. Modern FP8 inference is
*designed* to be near-lossless: the per-position perturbation it introduces is
comparable in magnitude to the honest numerical difference between two *correct*
implementations on different GPUs.

So the first honest finding is negative:

> **A per-position tolerance ε does not exist for the FP8 case.** Honest
> floating-point divergence and FP8 fraud are both "small perturbations of the
> correct computation." They overlap. Any design that checks one position against a
> threshold is broken, and we say so here to forestall it.

What rescues the scheme is a single asymmetry. Honest numerical noise is **roughly
unbiased** — a correct implementation does not systematically push the output
distribution in one direction. Quantisation is a **biased** estimator: it degrades,
consistently, in the same direction; it never improves perplexity. Therefore the
right object is not a per-position distance but a **sequential test on the
accumulated bias** over many challenged positions.

---

## The sequential test

The commitment is over the entire logit trace. A challenge opens a subset of
positions. At each opened position the verifier independently recomputes the
reference next-token distribution (with its own copy of M, under the canonical
numerics defined below) and forms a **bounded** per-position score
`X_j ∈ [0, 1]` (a clipped distributional discrepancy).

We deliberately do **not** model the scores as i.i.d. with known variance — they are
neither. We state the hypotheses on *conditional means* instead:

```
H0  (honest):  E[ X_j | F_{j−1} ]  ≤  μ0          (no independence assumed)
H1  (fraud) :  E[ X_j | F_{j−1} ]  ≥  μ0 + Δ
```

`μ0` is the honest-noise floor — a calibrated protocol constant (see *What we have
not figured out yet*). The test is constructed by betting. With a predictable bet
`λ_j ∈ [0, 1/(1−μ0))` chosen from data strictly before position *j*:

```
M_n  =  ∏_{j=1}^{n} ( 1 + λ_j · ( X_j − μ0 ) ) ,   M_0 = 1
```

Under H0, `E[ 1 + λ_j(X_j−μ0) | F_{j−1} ] = 1 + λ_j·( E[X_j|F_{j−1}] − μ0 ) ≤ 1`,
so `M_n` is a non-negative supermartingale. By **Ville's inequality**:

```
under H0:   P( ∃ n :  M_n ≥ 1/α )  ≤  α
```

Slash when `M_n ≥ 1/α`. The Type-I error — wrongly slashing an honest peer — is at
most α **uniformly over every sample size and every stopping rule, with no
independence and no variance assumption.** This matters more than anything else in
this document: the error the colony cannot afford, the one that would betray the
manifesto, is the one placed on the most rigorous footing.

The cost of catching fraud (the analogue of "how many positions must we open"):

```
E[ positions | fraud ]  ≈  (2 / e²) · log(1/α) · τ
        e = Δ/σ   (systematic bias / honest-noise std — the effect size)
        τ = integrated autocorrelation time of the score along the trace
```

Two consequences. First, `τ` is not fate: it is **set by the challenge-sampling
schedule.** Contiguous blocks → large τ. Positions spread across the trace → τ → 1.
**Spread sampling is therefore a derived protocol requirement, not a preference.**
Second, everything turns on `e`, and `e` is empirical. We return to this.

---

## The economics of when this works

A challenge of `m` positions, with KV-cache reuse over the audited prefix, costs
`≈ (m/L)·C_inf`, where `L` is the answer length and `C_inf` the inference cost. The
network challenges a fraction `p` of answers. Two constraints pin `p`:

- **Deterrence** (fraud must be −EV): `p > G / ((1−β)·S_NCS)`, where `G` is the
  compute the fraud saved and `S_NCS` the value of the standing destroyed by a
  slash (RFC-0001 / RFC-0003).
- **Budget** (verification overhead ≤ `ρ`): `p ≤ ρ·L / m`.

A challenge rate exists iff `p_min ≤ p_max`. Solving for the effect size yields the
viability threshold in closed form, in the primitives of RFC-0001:

```
                 ┌                                     ┐ ½
   e  ≥  T  =     │   2 · τ · G · log(1/α)             │
                 │  ───────────────────────────────── │
                 └   ρ_budget · L · S_NCS              ┘
```

`e ≥ T` → Tier 1 is economically defensible for that fraud class. `e < T` → it is
not, and that class falls back to Tier 2/3. **This is per fraud class and per
regime, not all-or-nothing.**

### The one lever we control before measuring

`σ` — the honest-noise std — is large only because honest peers run heterogeneous
kernels. The protocol can mandate a **canonical numerical recipe** (fixed reduction
order, deterministic kernels, possibly integer accumulation) **for the committed
path only**. The agent still serves the user its fast answer on whatever kernel it
likes; only the *audited* trace must follow the canonical recipe. This collapses σ
toward the true hardware-rounding floor, raising `e` — possibly from below `T` to
above it.

The price, stated with its teeth: **verifiability costs you the right to use your
fastest kernel on the path that gets audited.** This is the "heterogeneity tension"
of the manifesto, made concrete and quantified. RFC-0002 owns this tax; it does not
hide it. Which recipe, at which tax, is the central empirical question of b2.

---

## The challenge must be unforgeable

The sequential test is only as good as the challenge. The statistical seam
decomposes into **three** cryptographic requirements, not one:

```
R1  Binding-before-challenge   B is irrevocably bound to the WHOLE trace
                               before the challenge set S exists
R2  Unpredictable & non-       B can neither predict nor *grind* S
    grindable challenge
R3  Unbiasable auditor         B cannot choose, be, or foreknow its auditor
```

**The grinding trap.** The elegant non-interactive option — derive the challenge
from the commitment itself, `S = PRF(H(commit))` — is **insecure**. B controls part
of the preimage and can grind it: try variants until `S` lands only on the positions
it computed honestly. The lesson is load-bearing:

> Challenge randomness must come from public entropy that is **posterior to** and
> **unbindable by** B's commitment. There is no safe non-interactive shortcut.

The sub-protocol:

```
Inference (round r):
  B computes y; builds a Merkle tree over leaves  L_i = H(dist_i ‖ i)      [R1]
  root = MerkleRoot
  B publishes  commit = (root, H(M), x, req_nonce, agent_measurement, r)

Beacon (round r+1):
  B_{r+1} = an unpredictable public value, released AFTER r

Public, deterministic derivation — no verifier discretion:
  audited?       =  PRF(B_{r+1} ‖ root ‖ "sel")  <  p·2^k
  challenge S    =  Expand( PRF(B_{r+1} ‖ root ‖ "pos"), m, L )            [R2]
  assigned V     =  ReputationSet[ PRF(B_{r+1} ‖ root ‖ "aud") ]  (k-of-n) [R3]

Open:
  B reveals { dist_i, path_i : i ∈ S };  V recomputes p_i (canonical);
  betting e-process;  M_n ≥ 1/α  ⇒  slash via RFC-0003
```

- **No prediction / no grinding:** `root` is binding *before* `B_{r+1}` exists.
- **No equivocation:** Merkle binding forces consistent opening on the beacon's `S`.
- **No grief by a malicious V:** Ville holds for *any* `S` and *any* V, so an honest
  B passes with probability ≥ 1−α regardless; a malicious V can only fail to report
  a true fraud — and the opened leaves are public, so **a V that clears a provable
  fraud is itself slashable.** Verification is transitively verifiable.

### Beacon options

| Option | Strength | Honest cost |
|---|---|---|
| **(a) Ledger-derived** `H(RFC-0003 block_{r+1})` | ANTS-native, no new trust, no token | couples liveness to RFC-0003 cadence; the block must itself be unpredictable and non-grindable |
| (b) Threshold-BLS beacon, rotating high-reputation committee | strong unpredictability under honest threshold | a committee = mild centralisation the manifesto will scrutinise |
| (c) VDF on a public seed | no committee; non-grindable by slowness | VDF compute; still needs a seed B cannot control |

**We recommend (a)** — smallest surface, reuses an existing layer — with the
explicit caveat that it transfers a sub-problem into RFC-0003. (b) and (c) are
fallbacks if RFC-0003's cadence or entropy proves insufficient.

---

## Where this bottoms out

This sub-protocol does **not** dissolve the Sybil problem. It localises it. Tier-1
security reduces to exactly two named assumptions:

1. a **beacon** that is unpredictable, non-grindable, and live — handled above;
2. an **honest majority of the reputation/verifier set** (for R3, anti-collusion) —
   which is exactly **[RFC-0004](./RFC-0004-identity.md) (identity / Sybil
   resistance)**.

The dependency chain is now explicit and we write it down rather than hide it:

```
economy (RFC-0001) → verification statistics (this) →
verification cryptography (this) → ⊥ identity / Sybil (RFC-0004)
```

This is the same wall RFC-0001's footnote gestured at. The contribution of this
document is not to break it — it is to show, through exactly which joint, the whole
architecture rests on it. Cryptography has done its job here: it has narrowed
everything to one foundational assumption, named and no longer buried. **The real
load of ANTS bottoms out at identity, not economy.** RFC-0004 is therefore not the
fourth document. It is the foundation the other three stand on.

---

## What we have not figured out yet

If you are about to open an issue, here is where we want sharp eyes most:

- **`μ0` and the safety null.** The whole Type-I guarantee assumes honest
  heterogeneous hardware actually satisfies `E[X_j|F_{j−1}] ≤ μ0`. If some honest
  backend is *systematically* biased — a correct but skewed kernel — the null is
  false and honest peers are slashed. This is the catastrophic failure mode, worse
  than "`e` too small," and it is an empirical question we cannot answer at day
  zero. b2 must falsify it *before* anything else.
- **`e` is unmeasured.** The entire viability call is `e ≥ T`, and `e` for the FP8
  case under a real canonical runtime is not known. The literature suggests it is
  near the threshold, which is precisely the uncomfortable place.
- **`S_NCS` is endogenous.** The slash value `T` depends on is itself an open
  question of RFC-0001 (ledger portability). The two RFCs are coupled here; a change
  there moves the threshold here.
- **The canonical recipe is undecided.** Which numerical recipe, at which
  verifiability tax, is a Pareto question b2 must answer, not an a-priori choice.
- **The seam is honest but unproven at scale.** R3 holds only under honest-majority
  of the verifier set; that is RFC-0004's burden, and RFC-0004 does not yet exist.

---

## What comes next: b2, pre-registered

The verification layer is specifiable today but **not testable today** — it needs a
testnet, an implemented canonical runtime, and genuinely multi-vendor hardware. What
we commit to now is the *experiment and its decision rule*, fixed in advance so the
threshold cannot be retrofitted to produce a convenient answer:

```
GATE 1 — SAFETY (must pass first):
  Across ≥3 real backends × ≥2 attention kernels, every honest input-entropy
  regime satisfies the conditional-mean null with margin.
  Fails ⇒ Tier 1 is unsafe in that regime ⇒ TEE only.   [primary finding]

GATE 2 — VIABILITY (only if 1 passes):
  ∃ a canonical recipe R* with tax ≤ budget such that, for every adversary
  strategy with G ≥ deterrence-min,
        LCB( e | R*, strategy, regime )  ≥  UCB( T | RFC-0001 params ).
  True ⇒ Tier 1 GO in that regime ; False ⇒ TEE fallback in that regime.
```

`σ` is measured from the *worst plausible honest pair*, not the average — an honest
peer on exotic hardware must never be slashed. `Δ` is measured against the
*adaptive* adversary, including one that dithers to mimic honest variance: an
adversary forced to dither `Δ → 0` has destroyed its own quality or spent the
compute it tried to save, which is a win. Estimation is by block bootstrap, because
the data is heavy-tailed and correlated, and the decision is on confidence bounds,
never point estimates. The output is a pre-registered table —
`(model size × input-entropy regime) → { GO via R* | TEE fallback }` — and that
table is the empirical opening section this document will gain at v0.2.

A per-regime split is an expected and acceptable outcome, not a failure: Tier 1 may
be GO for factual queries and TEE-fallback for high-entropy ones. An honest,
actionable result beats a uniform claim we cannot defend.

---

## A reference sketch

In the habit of RFC-0001 — small enough to argue with, not production-grade:

```python
def audit(answer, commit, beacon_next, M_ref, alpha=1e-6, mu0=MU0, cap=CAP):
    # Public, deterministic — anyone can recompute this. No verifier discretion.
    if prf(beacon_next, commit.root, b"sel") >= commit.p * (1 << 64):
        return "not-audited"

    S = expand(prf(beacon_next, commit.root, b"pos"), commit.m, commit.L)

    capital, mu_hat, var_hat, n = 1.0, mu0, 1.0, 0
    for i in spread(S):                       # spread schedule → tau -> 1
        dist_i, path_i = answer.open(i)       # Merkle-bound; cannot equivocate
        assert merkle_ok(path_i, dist_i, commit.root, i)

        p_i = M_ref.reference_distribution(answer.prefix(i))   # canonical numerics
        x   = min(discrepancy(p_i, dist_i), 1.0)               # bounded score

        lam = clip((mu_hat - mu0) / var_hat, 0.0, cap)         # predictable bet
        capital *= 1.0 + lam * (x - mu0)
        if capital >= 1.0 / alpha:
            return "FRAUD"                    # Ville: P(this | honest) <= alpha

        n += 1
        mu_hat  += (x - mu_hat) / n
        var_hat += ((x - mu_hat) ** 2 - var_hat) / n

    return "passed"   # not "honest" — only: not caught this time
```

The last comment is the whole philosophy. The network never proves a peer honest. It
makes dishonesty unbluffable and unprofitable, and it remembers.

---

## How to contribute to this document

Open an issue on `Ants-Community/ants` with the tag `rfc-0002`. The same three paths
as RFC-0001: typo-and-clarity PRs (merged quickly); design-question issues
(discussed, then resolved with a PR or escalated to Matrix); proposed *amendments*
drafted as patches against this file with rationale.

The two places we will most defend, and most want broken: the claim that Type-I is
rigorously controlled without independence assumptions, and the claim that the
dependency chain genuinely bottoms out at RFC-0004 and nowhere shallower.

The document is CC0. Quote it. Disagree with it in public. Fork it.

---

*Drafted by the ANTS founding contributors, May 2026.
The colony has no center. This document does not either.*
