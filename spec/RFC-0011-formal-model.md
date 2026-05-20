# RFC-0011 — Formal Model of Cross-Layer Composition

**Status:** Draft · v0.1
**Topic:** The specification of what a formal model of the ANTS protocol must capture — invariants, threat model, compositions across the four time-scale layers. This RFC is the *intent* of the formal verification; the TLA+ (or equivalent) artefact that proves it is a downstream engineering deliverable.
**Audience:** You, if you write distributed-systems formal models, or are evaluating whether to trust this one.
**Depends on:** RFC-0001, RFC-0002, RFC-0003, RFC-0004, RFC-0005, RFC-0008, RFC-0009, RFC-0010

---

## Why this RFC, why now

The external multi-persona review of 2026-05-20 named the absence of a
formal cross-layer composition model as the single most expensive missing
piece short of the reference client itself. Quote:

> *"The composition 'DHT cache + CRDT slashing + PoUH chain + drand beacon'
> is a distributed ecosystem with four different consensus time-scales
> (eventually-consistent / monotone / 30s finality / round-bounded). I have
> not seen a formal analysis of how these compose under adversarial load.
> A formal model (TLA+ or ApPCS) before the PoC would be valuable."*

The criticism is correct. Each design RFC reasons about its own layer
honestly. None reasons about the **composition** of layers under
adversarial conditions at multiple time-scales. The pattern that has broken
real distributed systems (Solana partition recovery, Ethereum reorg
attacks, Bitcoin selfish-mining variants) is almost always
*composition-level*: each component is correct in isolation; the failure
mode lives at the seam.

This RFC does **not** write the formal verification. It **specifies what
the formal verification must capture** — invariants, threat model,
composition properties — so that the verification work, when commissioned,
has a clear deliverable. The verification itself is a downstream
engineering artefact, probably ~6–12 weeks of work by a TLA+-fluent
contributor against the spec below.

---

## What this RFC does NOT do

To prevent misunderstanding:

- It does **not** write the TLA+ code (or PRISM, ApPCS, Coq). That is a
  separate engineering deliverable.
- It does **not** prove the invariants. The verification work proves
  them.
- It does **not** declare which invariants are essential vs nice-to-have
  — all stated invariants are intended to hold; the verification work
  decides which can be feasibly proved and which require additional
  protocol clarification.
- It does **not** specify how the verification artefact integrates with
  the reference client (e.g., as a CI step, as a release-gate document,
  as a research paper). That is an `IMPLEMENTATION.md` concern.

---

## The four layers and their interactions

ANTS has four primary time-scales operating concurrently:

| Layer | Time-scale | Coordination property |
|---|---|---|
| **DHT + cache** | eventually consistent, no global state | Each peer holds slightly different views; resolution is local per query |
| **CRDT (L1)** | monotone append, gossip-bounded | Anyone holding a self-authenticating proof π converges to the same conclusion under propagation T_prop |
| **PoUH chain (L2)** | 30s block + 2/3 finality per epoch | Once 2/3 signs, block is final and globally agreed |
| **Drand beacon** | round-bounded external entropy | All peers see the same round value at the same round number |

Each operates correctly in isolation. The interactions:

1. **L1 × L2** — CRDT proofs propagate; L2 committee includes Merkle root over proofs visible at cutoff. Constraint: `T_prop < T_beacon`.
2. **L1 × L1 (pruning)** — Proofs included in L2 epoch summary `E_n` become prunable after `E_(n+1)`. Constraint: late joiners can still verify any historical slash via L2 Merkle root + archive-node retrieval.
3. **L2 × Drand** — Drand round value seeds VRF for committee selection; drand outage handled by degraded-seed flag. Constraint: drand-degraded blocks remain finalisable but are visibly marked.
4. **L2 × DHT** — Committee members are attested DHT peers; committee selection deterministically derives from VRF over the attested population.
5. **All × Bonds** — Bond admission is signed by a verifier and gossiped to L1; race-safe protocol via δ_admission + BLAKE3 tie-break on epoch_seed.
6. **Tier 3 committee × Bonds × Chain** — Tier 3 committee selection uses beacon-derived seed; each committee member's participation is bond-gated; bond admissions per Tier 3 act are L1 events.
7. **Verification × Cache** — Y_canon is committed at inference time and cached; Tier 2 audits recompute against Y_canon byte-for-byte.

The formal model must capture each of these interactions individually and
their composition under simultaneous adversarial conditions.

---

## Invariants to prove

Ten invariants, organised by layer and labelled for the verification work:

### Safety invariants

**I1 — Slash safety.** No honest peer applies a slash for a peer X unless
some honest peer at some past time held a valid fault proof
`π` such that `VERIFY(π, X.pubkey) = valid`.

**I3 — CRDT convergence.** Under any partial-honest gossip topology where
the honest subgraph is connected, the proof set held by every honest peer
converges to a superset of the proof set held by the slowest honest peer's
T_prop neighbourhood. (Equivalent: the G-Set is a CRDT under the standard
join semantics.)

**I4 — Chain finality.** No two conflicting blocks can both reach 2/3
committee signature without ≥ ⌈2K/3⌉ + 1 equivocating validators (each
of which is provably attributable per the equivocation slash). With K=64,
this is ≥43 simultaneously equivocating validators.

**I6 — Bond no-double-lock.** For any peer P at any time t,
`Σ over locked_bonds(P, t) ≤ A(P, t)`. No combination of race conditions
or verifier admissions can violate this invariant.

**I7 — Selective disclosure soundness.** A verifier admits a bond of
amount b iff the disclosed receipt subset S has
`Σ over r in S of decayed_value(r, t) ≥ b` AND every receipt in S is
countersignature-valid AND every Merkle inclusion proof verifies against
the peer's published `bag_root`. The verifier never admits a bond against
a false receipt or an inflated A claim.

**I8 — Canonical numerics determinism.** For any input x and canonical
model M, two honest implementations of the canonical recipe (RFC-0009)
produce bit-identical Y_canon. (This is a hardware claim partly outside
the formal model; the verification work specifies what the model assumes
about hardware and what would invalidate the assumption.)

**I9 — Y_canon caching invariance.** Every cache retrieval for the same
cache entry returns byte-identical Y_canon regardless of which peer
serves it.

### Liveness invariants

**I2 — Slash liveness.** A valid fault proof π generated at time t reaches
every honest peer by time t + T_prop, under the anti-eclipse assumption
that the honest subgraph contains at least one path of length ≤
T_prop / Δ between every pair of honest peers.

**I5 — Partition recovery convergence.** Under any network partition where
the partition heals before any single fork accumulates more than 1 epoch
of irreconcilable history, the protocol's Σ-T fork choice produces a
canonical reconciled state. Under longer partitions where both forks have
Σ T > θ · Σ T_total, the social-Schelling fallback is invoked
deterministically.

### Architectural invariants

**I10 — Genesis sunset.** Φ_genesis(t) → 0 monotonically as t → ∞ under
the bootstrap phase transitions, provided non-trustee `W_h` grows at a
rate exceeding `δ_genesis`. The "self-securing condition" of RFC-0010
§The self-securing condition is reachable under nominal parameters.

---

## Threat model

The formal model must verify the invariants under the following adversary
classes operating in combination:

| Class | Capability | Bound |
|---|---|---|
| **Byzantine peers** | corrupt up to fraction f of attested population | f < 1/3 for L2 finality; vertex-cut around victim for L1; honest-majority for PoUH committee selection (RFC-0005's Sybil cost is the economic bound) |
| **Network adversary** | partition, delay, drop, reorder messages | partition durations bounded by what social-Schelling fallback can tolerate; eclipse bounded by RFC-0005 anti-eclipse peer selection |
| **Timing adversary** | drand outage, local clock skew | drand outage handled by §4.3 degraded-seed; clock skew by race-safe bond admission |
| **Compute adversary** | cloud-TEE Sybil rental | bounded by RFC-0005 economics ($7M/yr per 10K identities at 2026 prices, scaling with network growth) |
| **Hardware adversary** | TEE vendor compromise | one vendor at a time per RFC-0005 multi-vendor weighting; bounded by attestation revocation propagation timing |
| **Patient adversary** | multi-year Sybil farming + cloud-TEE rental | bounded by A-as-bond mechanic per RFC-0004 v0.3 (`A` cannot be accumulated in advance) |
| **Social adversary** | community apathy, fragmentation | unbounded — the irreducible residual handed to the credible-fork-threat |

The formal model verifies invariants under combinations: e.g., a Byzantine
peer in a partial-partition with drand-outage and clock-skew. The
verification scope explicitly **excludes** the social adversary — that is
not a verifiable property; it is the named coordination assumption.

---

## Composition properties (the seams)

For each pair of layers, what must be verified that does not follow from
the single-layer invariants:

### L1 × L2 (CRDT × chain)

- **C1**: `T_prop < T_beacon` is satisfiable for realistic parameter
  ranges (gossip fanout f, network N) under the anti-eclipse assumption.
- **C2**: L2 epoch summary Merkle roots commit to L1 proofs that have
  fully propagated. Pruning after epoch confirmation does not create
  unverifiable slashes for late joiners (the L2 root + archive-node
  retrieval suffices).

### L1 × DHT

- **C3**: Anti-eclipse peer selection at the DHT layer produces a graph
  in which the honest subgraph remains connected under realistic Sybil
  attack fractions per RFC-0005.

### L2 × Drand

- **C4**: VRF committee selection from drand-degraded seeds remains
  predictable to honest verifiers but not to attackers who control less
  than 1/3 of the previous committee. (This is the residual security
  regression during outages, named explicitly.)

### L2 × DHT

- **C5**: Committee selection is reproducible from the L2 chain state +
  drand round; a peer verifying historical commitments deterministically
  recomputes which committee signed which block.

### All × Bonds

- **C6**: Bond admission protocol is race-safe: no peer's
  `locked_total > A` at any moment, under any sequence of race conditions
  among verifiers.

### Tier 3 × Bonds × Chain

- **C7**: A Tier 3 verification act requires both committee selection
  (chain-derived) and bond admission (per-peer A-gated). No combination
  of these admits a malicious committee without ≥ ⌈2N/3⌉ peers each
  bonding for the act AND each colluding.

### Verification × Cache

- **C8**: Y_canon caching invariance (I9) holds across the cache write
  protocol and the cache hit protocol: writes commit Y_canon, hits return
  Y_canon, Tier 2 audits recompute Y_canon. The cache is a CRDT under
  Y_canon keys (no two writes to the same key can disagree about the
  cached value because both must produce the canonical recipe output).

---

## Recommended tooling

The verification work has at least three viable paths:

1. **TLA+** (Lamport, with TLC and Apalache model checkers) — the gold
   standard for distributed protocol safety/liveness. Strongest tool for
   I1–I7, C1–C7. Maturely supported, well-tutorialised. **Recommended as
   the primary verification artefact.**

2. **PRISM** (probabilistic model checker) — better suited for I8
   (canonical numerics statistical claims) and the e-process Type-I/II
   bounds. Complements TLA+ rather than replacing it.

3. **Coq / Lean** (theorem provers) — overkill for the invariants here
   but useful if the project wants machine-checked proofs of the
   canonical-numerics determinism (I8) at the hardware level. Probably
   future work.

The recommended sequence: TLA+ specification of layers (I1–I7) and their
compositions (C1–C7) first; PRISM verification of e-process statistical
claims separately; Coq/Lean as a v2.0 deliverable if the project's risk
appetite warrants the cost.

---

## Verification scope: what counts as "done"

The TLA+ artefact, when delivered, must satisfy:

- **All ten invariants** specified, with model-checked verification at
  representative parameter scales (e.g., K=8 committee, N=20 peers, f=5
  Byzantine — large enough to exercise the composition, small enough that
  TLC terminates).
- **All eight composition properties** specified, with at least one
  representative scenario for each.
- **Counter-examples generated and resolved** for any invariant that
  initially fails. If an invariant cannot be proven, the spec is amended
  until it can — that is the *purpose* of the verification work.
- **Refinement mapping** to the actual implementation (so that a critical
  bug in the TLA+ spec maps to a specific protocol RFC and a specific
  implementation component).

The PRISM artefact, when delivered, must satisfy:

- I8 honest-noise floor calibrated probabilistically against the
  canonical recipe (this is partly empirical, requires b2 measurements as
  input).
- E-process Type-I bound α verified for realistic parameter ranges (μ_0,
  Δ, σ_estimator).
- Detection-time bounds for fraud classes of interest (FP8-as-BF16,
  smaller-model substitution, replay, etc.).

A passing TLA+ artefact + a passing PRISM artefact, together, constitute
"the formal model" referenced by GOVERNANCE.md as the v1.0 condition's
"specification stability for 6 months" — both must hold at the version of
the spec being assessed.

---

## What we have not figured out yet

- **Exact mapping from TLA+ refinement to implementation components.** The
  reference client's sub-component decomposition (`IMPLEMENTATION.md`)
  must point to which TLA+ module a given component refines.
- **Parameter scale at which TLC terminates.** Each composition has its
  own state-space; the verification work picks representative scales but
  may have to use Apalache symbolic checking for larger parameters.
- **Whether PRISM is the right tool for the e-process.** Anytime-valid
  betting processes are a relatively new construct in probabilistic
  verification literature; PRISM may need encoding tricks. Alternative:
  Bayesian posterior verification.
- **Adversary models in TLA+ with realistic timing.** TLA+ has temporal
  logic but realistic-timing adversaries (clock skew, drand outage
  duration distributions) require Apalache or PRISM additions.
- **Cost.** A first-class TLA+ verification artefact is 6–12 weeks of
  work for a TLA+-fluent contributor. The project does not yet have such
  a contributor identified.

---

## How to contribute

Open an issue on `Ants-Community/ants` with the tag `rfc-0011`. Three
kinds of contribution most valuable right now:

- **TLA+ contributors willing to write the spec.** This is the largest
  outstanding deliverable in the corpus after the reference client and
  the canonical kernel library.
- **PRISM contributors for the e-process verification.** Smaller scope
  than the full TLA+ work but technically deep.
- **Reviewers of this specification.** If an invariant is missing, or a
  composition property is mis-stated, the spec needs to know before the
  verification work begins.

The document is CC0. Adopt the formal-model-of-intent pattern for your
own protocol if helpful.

---

*Drafted by the ANTS founding contributors, May 2026.
A model that is not formal is a guess. A model that is formal but not specified is a different guess.*
