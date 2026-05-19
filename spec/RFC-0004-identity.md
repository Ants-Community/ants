# RFC-0004 — Identity

**Status:** Draft · v0.1
**Topic:** The identity layer of ANTS — why a peer is one peer, and what the whole architecture rests on.
**Audience:** You, if you are willing to disagree in writing.

---

## Who is the peer in Trento?

In [RFC-0001](./RFC-0001-barter.md) an answer came back from a peer in Trento.
In [RFC-0002](./RFC-0002-verification.md) you learned how to tell that the answer
was not a lie.

This document asks the question both of those quietly assumed: **how does the
network know the peer in Trento is one peer, and not ten thousand sock-puppets
owned by the same adversary** — one of which served you, the other nine thousand
nine hundred and ninety-nine of which sit in the verifier set, ready to vouch for
each other's lies?

RFC-0002 proved its guarantees *conditional on an honest majority of the verifier
set*. RFC-0001's economy assumed verification worked. Both delegated their trust
downward. This is the document they delegated it to. There is nothing below it.
It is, therefore, the most important document in the project and the one we are
least able to make trustless. We will be honest about exactly why.

---

## What we are trying to prevent

A Sybil attack: one entity manufacturing many identities to acquire, cheaply, a
fraction of the network's verifier/reputation weight large enough to break the
honest-majority assumption RFC-0002 (its anti-collusion requirement, R3) depends on.
Call that tolerated fraction **θ**. Everything in this document is about keeping the
adversary's effective weight below θ — using only what the manifesto permits.

---

## The cage we built for ourselves

We must be honest before we are clever. The three mechanisms the rest of the world
uses to resist Sybil attacks are **all forbidden by our own manifesto**:

- **Proof-of-stake / economic bond** — the single most effective permissionless
  Sybil defense. Forbidden: RFC-0001 refuses tokens and money.
- **Proof-of-personhood** — biometrics, phone, government ID, social graph.
  Forbidden: refusal #13. "Identifying our users" is precisely what we will not do.
- **A trusted issuer / certificate authority** deciding who is real. Forbidden:
  refusal #14. No central authority over the protocol.

This layer is the hardest in ANTS **by the manifesto's own deliberate choice.** That
is a cost we have chosen, not a free lunch, and the manifesto should say so.

It also means one sentence in the project's README is the weakest claim in the whole
corpus:

> *"Identity — proof of unique hardware. One CPU, one identity. No KYC. Sybil
> attacks must rent thousands of physical chips, not register thousands of
> accounts."*

This is not true as stated. Cloud TEEs are rentable attested chips at scale;
renting ten thousand of them costs far less than the value of controlling a
sovereign network's verifier set. "Genuine chip" attestation is signed by a chip
vendor — a central authority, exactly the one refusal #14 rejects, merely
relocated, and one you cannot fork away from. And a hardware key unique enough to
stop Sybil is unique enough to track the user, contradicting the same refusal #13.
RFC-0004 **corrects the README**, in the project's habit: this is where slogans meet
honest engineering (RFC-0002 did the same to the manifesto's "every machine is an
equal verifier"). Proof-of-unique-hardware is, at best, an *auxiliary* cost-raiser.
It is not the foundation. The foundation is built differently, and lower.

---

## Why there is no clean answer

| Mechanism | Status in ANTS | Why it does not stand alone |
|---|---|---|
| Proof-of-stake | **forbidden** | the robust answer; removed by our own refusal of money |
| Proof-of-personhood | **forbidden** | "identifying users" — the refusal #13 line |
| PoW identity rate-limit | weak | an arms race the richest adversary wins; taxes the laptop in Naples |
| Hardware / TEE attestation | auxiliary only | rentable cloud at scale; vendor = central authority; uniqueness↔privacy; excludes consumer machines; inherits vendor breaks |
| Exotic physical (PUF, proof-of-location) | auxiliary only | needs an enrolment authority or is spoofable |
| **Reputation-weighting fused with the economy** | the coherent direction | but recursive with RFC-0002 — addressed below |

So, as in RFC-0002, we do not pick one mechanism. We invert the problem.

---

## The thesis: amortised identity

Stop trying to prove hardware uniqueness — it is impossible cheaply inside the cage.
Instead: **identity is free to create, and worthless until it has been amortised by
verified useful work.** Ten thousand fresh identities have zero verifier weight
until each has independently earned reputation by doing real, audited, useful work —
which is exactly the work the network wanted in the first place.

The adversary needs `W_a ≥ θ/(1−θ)·W_h`, where `W_h` is honest earned weight. There
are only two ways to get it:

- **Path 1 — earn it for real, under many identities.** This *is* useful work; to be
  positioned to attack, the adversary must first become one of the largest honest
  contributors. And because reputation **decays** (RFC-0001 already chose this: "*
  reputation is recent behaviour, not accumulated wealth*"), weight is not bought
  once — it must be *sustained*. The moment the adversary stops doing honest work to
  cash in the attack, its weight decays and it falls below θ.
- **Path 2 — fake the work.** This is precisely the fraud RFC-0002 is built to make
  −EV. Path 2 is closed *if and only if RFC-0002 is sound* — which holds *if and
  only if* the verifier set is honest-majority, which is what this document is trying
  to ensure. **The recursion between RFC-0002 and RFC-0004 lives exactly in this
  choice between Path 1 and Path 2.**

In plain numbers, the attack is irrational while

```
   c_a · (θ/(1−θ)) · ρ_h · τ_hold   >   Payoff ≈ θ · Φ · G · (1/δ)
```

`c_a` adversary unit-cost, `ρ_h` honest work-rate, `Φ` network inference rate, `G`
fraud gain (RFC-0002), `δ` reputation decay. The cost of *holding* position grows
linearly with time; the payoff is **capped by the decay-limited exploit window
`1/δ`**. For fast enough decay there is no profitable window.

The closure worth stating: **RFC-0001 introduced reputation decay for choke-loop
responsiveness. That same decay is, unintentionally, the primary Sybil defence.**
Sybil resistance was hiding inside the economic layer's decay parameter all along.

Note also that `W_a` and `W_h` both scale with the network, so the attack-cost /
payoff ratio is **roughly independent of network size**. ANTS is not safer because
it is bigger. It is safer when decay is fast and adversary cost is near parity — and
it is *insecure at genesis*, because at genesis `W_h ≈ 0` and the ratio breaks.

---

## Where it bottoms out, part 1: the genesis bootstrap

The brutal honest statement first: **a permissionless network with no money, no
identity and no authority cannot bootstrap Sybil resistance from absolute zero.**
This is not a failure of ANTS; it is a known result wearing a disguise. Every real
decentralised system has a genesis trust root — Bitcoin's genesis block and the
social consensus around it, Tor's directory authorities (still there, never
retired), every proof-of-stake chain's initial validator set, Certificate
Transparency's trusted logs, the root CAs. There is no exception, and we will not
pretend to be one. Pretending would be the project lying to itself, which is the
one thing the manifesto's honesty exists to prevent.

The honest contribution is not the absence of a genesis trust assumption. It is
making that assumption **explicit, minimal, provably decaying, and
fork-recoverable** — which the systems named above mostly do *not*.

Concretely: a public, socially-anchored genesis trustee set (its keys committed in
the genesis block, its behaviour auditable by anyone), holding *granted* weight that
is **non-renewable** and decays at the same `δ` as everything else. Define
`G(t) = G(0)·e^{−δt}` and the genesis set's influence
`Φ_genesis(t) = G(t) / (G(t) + W_h(t) + W_a(t))`. Two requirements separate:

```
(D1)  Asymptotic retirement — EASY:
      G(t) → 0  while  W_h → ρ_h/δ > 0   ⇒   Φ_genesis → 0
      free under δ, provided honest work materialises (ρ_h > 0)

(D2)  The crossover before exploitation — HARD:
      t_safe = first t with  W_h ≥ (1−θ)/θ·W_a  AND  Φ_genesis < θ
      the bootstrap is sound  ⟺  the genesis set stays honest on [0, t_safe]
```

So genesis trust is not eliminated. It is **bounded to a finite interval
`[0, t_safe]`**, made an explicit decaying function, proven to vanish, and — the
part that does the real work — made recoverable by fork. This is *weaker* than "no
authority," and we say so. It is also *stronger* than what Tor, the PoS chains, and
the premined foundations actually deliver, because they never retire their genesis
trust at all.

---

## Where it bottoms out, part 2: the credible fork threat

What protects `[0, t_safe]` against a corrupted genesis set is one thing: a
**credible fork threat.** This is a deterrent, identical in philosophy to every
other deterrent in ANTS (choke/unchoke in RFC-0001, slashing in RFC-0002) — a
mechanism that exists in order *not* to be used. It constrains the genesis set's
behaviour because misbehaviour triggers the genesis set's own irrelevance.

"Credible" is not a soft word. It decomposes into three concrete, designable
preconditions:

- **Observability** — genesis misbehaviour must be detectable. Pre-RFC-0002 this is
  social and out-of-band, and so it *requires transparency* of the genesis set.
- **Coordinability** — the honest majority must converge on the *same* successor,
  leaderlessly. This needs a Schelling focal point: the manifesto and the public
  RFCs are not branding — for this layer they are load-bearing security
  infrastructure.
- **Low exit cost** — forking must be cheap. Here the non-obvious payoff: **a token
  would create a sunk financial cost anchoring people to the captured instance,
  making the threat non-credible. Refusal #12 (no token) is therefore a security
  property, not only an ideology.**

This reframes the manifesto. Its most "ideological"-sounding refusals are in fact
the *engineering* of fork-credibility: #12 (no token → low exit cost), #13 (no KYC →
no captured identities locking users in), #14/#15 (no central authority / "we will
fork against them" → the explicit Schelling commitment). **For the genesis layer,
the manifesto is not the philosophy wrapped around the protocol. The manifesto is
the security mechanism.**

And the caveat, because we do not oversell: this means ANTS's ultimate guarantee is
**not trustless** — it is trust-minimised and trust-transparent. It is exactly as
strong as the honest community's ability and willingness to observe and coordinate.
ANTS does not *eliminate* the centralisation problem; it *converts* it from a
technical problem (unsolvable inside our self-imposed cage) into a social one (made
maximally observable and recoverable, but not eliminated). Whether that trade is
worth making is the real question this document puts to its reader — it does not
assume the answer.

---

## Making fork-recovery precise

"The community can fork" is not a mechanism until it can be done **without an
authority declaring which fork is legitimate** — otherwise refusal #14 walks back in
through the recovery door. Five sub-problems, each made precise, each forcing a
protocol requirement:

1. **Legitimacy = a cryptographically attributable fault**, enumerated in advance,
   not an accusation or a vote (a vote is symmetric: a malicious actor can fork
   against an honest genesis set — fork-griefing). Equivocation is the gold
   standard: two conflicting signed statements *are* the proof, verifiable offline
   by anyone, no honest majority needed to adjudicate. **Requirement that follows:
   the genesis weight schedule and every grant must be public, signed, append-only
   commitments**, so over-issuance becomes attributable equivocation against the
   schedule. Residual, stated plainly: *censorship is not attributable* ("refused to
   include X" vs "X never arrived" are indistinguishable). A smart malicious genesis
   set censors rather than equivocates. This is the irreducible social weak spot.

2. **The successor is derived, not chosen.** Deterministically from the last agreed
   pre-fault state plus a pre-committed, time-phased succession rule: a pre-named
   backup-set *sequence* near `t ≈ 0` (which weakens the assumption from "trust this
   set" to "trust that at least one of a public sequence is honest, and abuse of
   each is attributable"), handing over to the top earned-reputation peers as `W_h`
   grows. The recovery's trust source thus **self-decentralises on the same δ/W_h
   curve** the genesis weight decays on. Pure social Schelling remains — but as the
   named, minimised, manifesto-anchored *ultimate* fallback, not a blank coordination
   problem.

3. **State portability.** If a fork resets honest earned reputation to zero, exit
   cost is catastrophic and the threat dies. So reputation is portable across a
   fork — *conditionally*: only the part verifiable against the pre-fault agreed
   state ports; the fault proof's timestamp defines the cut. (One consequence we
   record honestly: this **reprioritises RFC-0001's open "ledger portability"
   question from a v0.3 nicety to a load-bearing security parameter of this layer.**)

4. **Griefing symmetry.** Because the legitimacy trigger is a cryptographic fault
   proof, a false alarm produces no proof, no Schelling convergence, and the false
   fork simply dies. The same precision that enables real recovery also neutralises
   recovery's inverse abuse.

5. **The irreducible floor.** Even with all of the above, recovery works only if a
   majority of honest *weight* actually relocates. Nothing in the protocol can force
   this. We have reduced "trust the genesis set" to "*given a verifiable fault
   proof, a deterministic successor, and portable state, the honest majority will
   act*." The first three are now in-protocol and precise. The last is a
   **coordination assumption that cannot be made cryptographic** — the same one
   under weak subjectivity, social slashing, and the UASF precedent. ANTS does not
   escape it. It only drives the activation energy toward zero: trivially verifiable
   proof (trust no accuser), deterministic successor (negotiate nothing), portable
   state (lose nothing by moving). The floor is a *minimised but non-eliminable*
   coordination assumption, and we name it as the floor.

---

## The shared spine: there is no single δ

We have been calling one thing `δ`. That is the error. It conflates three decays
with opposite needs: choke responsiveness wants it fast (minutes); persistence and a
stable fork-succession set want it slow (weeks); the Sybil exploit window wants it
fast. The single-`δ` feasible region — fast enough for Sybil *and* slow enough for
persistence — is **empty**. (The apparent cold-start conflict is illusory: RFC-0001's
cold start rests on the optimistic-unchoke slot and the compute-floor, not on
decayed stock.)

The resolution is a **two-component reputation type, referenced identically by
RFC-0001, RFC-0003 and RFC-0004**:

```
   R = (A, T)
   A  "active"  — decays fast (δ_A): serving priority, Sybil-window bound,
                  genesis-weight retirement
   T  "tenure"  — decays slow (δ_T): persistence, stable fork-succession,
                  and acquired under a hard per-identity rate cap κ

   verifier ELIGIBILITY  ←  gated by T   (slow, stable)
   verifier WEIGHT / serving priority  ←  governed by A   (fast, volatile)
   an attack requires sustained-high A  AND  mature T,  simultaneously
```

`κ`, the tenure acquisition-rate cap, is the new primitive that finally delivers
what "one CPU, one identity" wanted and could not: **time as the resource that
cannot be bought.** Not hardware (rentable), not stake (forbidden) — a protocol cap
making trust-eligibility accrue at a fixed rate per identity that no amount of money
or compute can accelerate. *One identity, one clock; you cannot rent more clock.*

Two closures fall out, unforced:

- `κ` also caps a *legitimate* whale: the GPU farm in Reykjavík does 1000× the work
  but accrues tenure at the same capped rate. Tenure is **egalitarian by
  construction**, which resolves the legitimate-concentration (Gini) risk this
  document raised. The decoupling fixes two of this layer's three honest limits.
- Slow `δ_T` is only safe because a bad actor can be removed *faster than decay* —
  by attributable-fault slashing. That primitive from §fork-recovery now does
  **quadruple duty**: RFC-0002 transitive verifiability, fork legitimacy,
  portable-state cut, and tenure slashing. One primitive, four jobs — the structural
  economy the manifesto's "small surface area" asks for, emergent rather than
  imposed.

---

## ANTS is a single recursive object

This is the structural truth this document must state outright. Every step couples
the layers. RFC-0002's threshold pulled in RFC-0001's slash value. This document's
fork-recovery pulled RFC-0001's ledger portability from optional to load-bearing.
The shared spine is not a constant — it is the `(A, T, δ_A, δ_T, κ)` *type* every
other RFC consumes.

ANTS is **not four independent RFCs.** It is one recursive object: the social root,
identity, verification and economy stand or fall together. The reader who wishes to
attack the design should attack the recursion, not a single layer — and the reader
who wishes to defend it should understand that no layer is defensible alone.

---

## What we have not figured out yet

- **The cost-asymmetry wall.** Everything here makes Sybil −EV for an adversary at
  cost parity. A state- or cloud-scale adversary with `c_a ≪ c_h` is only *bounded*,
  not stopped. We do not have a defence inside the cage; we have a deterrent and an
  honest admission.
- **The patient pre-positioned adversary.** Farm tenure on cheap identities from
  genesis, do minimal work, wait for maturation. `κ` *delays* this; it does not
  prevent it. The cost becomes "you must have started years ago and sustained
  presence" — large, but not infinite.
- **`κ` is a new 1-D tension.** Too low and verifier eligibility grows too slowly
  (recursion with RFC-0002's need for enough honest verifiers); too high and the
  patient adversary is cheaper. We traded a structurally infeasible 3-way `δ`
  conflict for one isolated, tunable, measurable parameter. Progress — not a free
  lunch.
- **Censorship is not attributable.** The fork-legitimacy machinery has no
  cryptographic answer to a genesis set that censors rather than equivocates.
- **Social-layer capture.** The credible-fork-threat assumes a community capable of
  and willing to coordinate. Apathy, fragmentation, or capture of the social layer
  erodes the only floor the architecture has.

---

## What comes next

[RFC-0003](./RFC-0003-reputation.md) (Reputation) must specify the `(A, T)` type,
the rate cap `κ`, the decay rates, and the attributable-fault proof format that does
the quadruple duty above. RFC-0004 does not stand without it; nothing in ANTS stands
alone. That is the finding, not a caveat to it.

We state plainly what this document is: the foundation the other three lean on, and
the least trustless of the four — necessarily, by the manifesto's own design. ANTS
cannot be turned off *if and only if* its community remains able and willing to
fork. That is the whole claim. It is smaller than the manifesto's rhetoric, and it
is the true one.

---

## A reference sketch

In the habit of RFC-0001 and RFC-0002 — small enough to argue with, not
production-grade:

```python
def effective_weight(peer, now):
    A = peer.active_rep   * exp(-DELTA_A * (now - peer.a_ts))   # fast
    T = peer.tenure_rep   * exp(-DELTA_T * (now - peer.t_ts))   # slow
    return A, T

def eligible_as_verifier(peer, now):
    _, T = effective_weight(peer, now)
    return T >= TENURE_FLOOR            # eligibility gated by slow tenure

def accrue(peer, verified_work_units, now):
    peer.active_rep += verified_work_units                 # A: uncapped, fast-decay
    gain = min(verified_work_units, KAPPA * (now - peer.t_ts))
    peer.tenure_rep += gain                                # T: rate-capped by KAPPA
    peer.a_ts = peer.t_ts = now

def genesis_influence(now):
    return G0 * exp(-DELTA_A * now)      # non-renewable; retires on the fast clock

def fork_is_legitimate(claim):
    # NOT a vote. A self-authenticating proof, verifiable offline by anyone.
    return verifies_attributable_fault(claim)   # e.g. two conflicting signed msgs

def on_attributable_fault(peer):
    peer.tenure_rep = 0.0                # slow decay is safe because removal is instant
```

The shape of the whole protocol is in the last two functions: trust is earned
slowly and on a clock no one can rent, and it is destroyed instantly and by anyone
holding a proof. Everything else is parameter-tuning around that.

---

## How to contribute to this document

Open an issue on `Ants-Community/ants` with the tag `rfc-0004`. Same three paths as
RFC-0001 and RFC-0002: clarity PRs merged quickly; design-question issues discussed
then resolved or escalated to Matrix; amendments drafted as patches with rationale.

The two claims we will most defend, and most want broken: that amortised identity
makes Sybil −EV for any adversary at cost parity, and that the architecture
genuinely bottoms out at a *bounded, recoverable* social assumption and nowhere
shallower. If either is wrong, this is the document where it matters most.

The document is CC0. Quote it. Disagree with it in public. Fork it. The right to do
the last one is, as this document argues, the foundation everything else stands on.

---

*Drafted by the ANTS founding contributors, May 2026.
The colony has no center. This document does not either.*
