# ANTS RFC CHANGELOG

A record of substantive decisions and amendments to the RFCs. Per RFC-0001's
amendment process: clarity and typo PRs are not logged here; design-changing
amendments are. Each entry states what was, what is now, and why.

---

## RFC-0001 — Barter · v0.1 → v0.2 · 2026-05-19

Two amendments, both forced by discoveries in the later RFCs (RFC-0002 through
RFC-0004) once the full corpus closed. RFC-0001 was **not** rewritten; only these
two points changed.

### 1 · Ledger portability: open question → reasoned trade

**Was (v0.1):** an open question — *"each peer's view is private to it, and that's
deliberate… portable, attested credential? We lean toward 'OK for v0.1, revisit in
v0.3.'"*

**Now (v0.2):** reclassified from a deferred convenience to a **security
parameter**. RFC-0004's fork-recovery showed the credible-fork-threat — the
architecture's fixed point — fails if a fork resets earned standing, so portability
is load-bearing, not optional. The portable credential is RFC-0003's
selective-disclosure threshold proof (commitment to the receipt bag + a
zero-knowledge proof that tenure ≥ floor). It is sound because reputation is
asymmetric: positive evidence is holder-presented and selective disclosure can only
*under*-claim; negative evidence (slashes) is not in the holder's bag — it is
RFC-0003's non-suppressible global set — so faults cannot be hidden. Irreducible
cost: there is no zero-leakage portable reputation; the one bit "established vs
newcomer" is unavoidable, and it is exactly the signal cold start and identity
functionally require. Full portability and full ledger-privacy are not jointly
satisfiable; minimised-disclosure portability is the chosen trade.

### 2 · S_NCS: assumed parameter → derived quantity

**Was (v0.1):** undefined. RFC-0002's deterrence threshold needed "the value of the
standing a slash destroys" and flagged it as its weakest, endogenous input.

**Now (v0.2):** derived. A slash's durable cost is κ-rate-capped tenure re-accrual:
`S_NCS ≈ w · TENURE_FLOOR / κ` (a lower bound; conservative — the true deterrent is
stronger). `w`, the establishment premium, is the only residual and is measurable
on the testnet, not assumed. Consequence: `S_NCS ∝ 1/κ` ⇒ RFC-0002's threshold
`T ∝ √κ` ⇒ a smaller κ strengthens **both** Sybil resistance (RFC-0004) and
verification viability (RFC-0002). The two hard layers align in κ rather than
trading against each other: **κ is the single master security knob of the
architecture.** This closes the endogeneity RFC-0002 flagged; it does not close the
patient single-decisive-act residual.

---

## Architectural pivot · corpus v0.1 → v0.2 · 2026-05-20

A corpus-level restructure with several pieces. Recording it here because it is
exactly the kind of design-changing amendment RFC-0001's own process prescribes a
CHANGELOG entry for, and the pivot commit landed without one — a process gap this
entry closes.

The pivot has six pieces. The first three are structural — the cache, the economic
split, the renumbering. The last three are substantive design choices in
individual RFCs. The animating principle, applied to all six: where v0.1 had
derivations and proofs, those survive as the technical mechanism beneath a new
consumer-facing surface. No prior result is discarded silently. Where a prior
result is rejected, it is rejected by argument, here.

### 1 · RFC-0002 Semantic Cache · new layer

**Was:** no cache layer in the corpus.

**Now:** RFC-0002 (Semantic Cache) introduced as a foundational layer — a
distributed content-addressable store keyed by prompt embedding. Every honest
answer the network has given lives there, signed by its producer, retrievable by
similarity.

**Why:** The deepest economic critique of the v0.1 corpus identified a structural
pathology in pure barter: specialists whose expertise is rare and whose demands
are modest accumulate unspendable credit, because asymmetric demand cannot be
settled bilaterally. The cache resolves this directly: a high-quality answer
becomes durable yield rather than a one-time payment, paid in fractional NCS to
the original producer on every subsequent retrieval. It is the correct response to
the real problem, and it does so without introducing money, tokens, or central
coordination.

### 2 · RFC-0001 split into Community Economics + RFC-0006 Payment Terms · v0.2 → v0.3

**Was:** a single RFC-0001 (Barter) implying barter was the protocol's only
economy.

**Now:** RFC-0001 (Community Layer Economics, v0.3) specifies barter for the
community population. RFC-0006 (Payment Terms) specifies the metadata field by
which any peer declares its economic terms — barter, fiat, subscription, gift, or
custom — and the rules for coexistence. Multiple economies share the transport
layer, the cache, and the identity layer; they are isolated economically and
interoperable everywhere else.

**Why:** Another v0.1 critique flagged that "no money at the protocol layer"
combined with "cloud providers can sell access on top" silently recreates the
centralization the manifesto refuses, one layer up. The HTTP analogy applies: HTTP
does not know what payment is, and that neutrality is why it carries every
economy. The protocol declares no economy; populations declare their own; the
foundation captures none of them.

### 3 · RFC renumbering

**Was:** RFCs 0001–0004 covered Barter / Verification / Reputation / Identity.

**Now:** RFCs 0001–0006 cover Community Economics / Semantic Cache / Verification
/ Reputation / Identity / Payment Terms. RFC-0007 (Post-v1.0 Governance) is
reserved.

**Why:** the cache becomes architectural peer to the economy, not a sub-component.
Payment Terms peer to the economic layer rather than nested in it. Numerical
contiguity is a minor cost worth paying for structural clarity. The old files
under the prior numbering have been kept temporarily for diff and are scheduled
for removal in a follow-up.

### 4 · RFC-0003 Verification · v0.1(early) → v0.2 · marriage of tier framing and scheme (C)

**Was (v0.1 early):** a three-tier consumer-facing menu (attestation only /
cross-check / triangulation) with key parameters declared as needing calibration —
sampling rate 3%, cross-check cosine threshold 0.85, and so on — and no
anti-grinding analysis of the challenge mechanism.

**Was (in prior v0.1 derivation cycles):** a single-mechanism design — scheme (C),
commit-at-send · prove-if-challenged · lose-everything-if-caught — with: a
bounded-discrepancy aggregate sequential test (an anytime-valid betting e-process,
controlled by Ville's inequality with no independence or variance assumption);
the closed-form economic viability threshold `T = √[2τG·log(1/α) / (ρ·L·S_NCS)]`
relating effect size to the deterrence economics of RFC-0001; a binding
cross-layer constraint linking the slash propagation window to the verifier
rotation cadence; the canonical-numerics requirement on the audited path as the
pivotal design lever; and a pre-registered two-gate testnet experiment (the
"safety null first, viability second" protocol called b2 internally).

**Now (v0.2):** both, integrated. The three-tier menu is the outer, consumer-facing
structure — it stays, because it is the right way to expose a cost/assurance
tradeoff to the requester. The unified primitive beneath all three tiers is
commit-at-send: Merkle binding of the full logit trace, beacon-bound challenge
selection (anti-grinding), beacon-derived auditor assignment (anti-collusion).
Tier 1's "3% sampling" becomes a default *example* with its deriving inequality
made transparent (the sampling rate `p` is bounded below by the deterrence
condition and above by the budget, both derived from RFC-0001 economics). Tier 2's
"cross-check" is scheme (C) properly stated, with the betting e-process replacing
the per-position cosine threshold of the v0.1(early) draft. Tier 3's triangulation
has each of the N committee peers publish independent commit-at-send commitments,
so the anti-grinding guarantee is per-peer and the convergence test sits on top of
it. Canonical numerics is a hard requirement for any tier ≥ 2. The b2 experiment
opens the document as the empirical decision rule.

**Why:** the v0.1(early) draft introduced a much better user-facing surface than
any prior draft had — the explicit tier-by-tier menu is genuinely good design, and
it covers cases (Tier 3 triangulation, human feedback for subjective domains) that
the earlier single-mechanism design did not. But its parameters were guesses, and
its cross-check rule (per-position cosine) had been proven inadequate against the
make-or-break adversary class (FP8-degradation fraud, where honest hardware noise
and dishonest quantization overlap per-position and are separable only in
aggregate). The earlier work derived the rigorous mechanism that fixes the
parameters and the cross-check; it was published and survived independent review.
Discarding it and replacing the parameters with guesses would cede ground to
expert critique that the project had already earned. Integrating keeps both: the
explicable surface, the defensible mechanism. No funeral. A marriage.

### 5 · RFC-0004 Reputation · v0.1(early) → v0.2 · two layers, witness over CRDT

**Was (v0.1 early):** a single mechanism — a small slow blockchain with novel
"Proof-of-Unique-Hardware" (PoUH) consensus as the primary structure. Slashing
arises from finalized chain blocks. Hardware attestation provides Sybil
resistance; VRF selects committees of K=64 from the attested peer population;
2/3 committee signatures finalize a block.

**Was (in prior v0.1 derivation cycles):** a consensus-free design — a grow-only
set (G-Set CRDT) of self-authenticating monotone fault proofs. `VERIFY(π)` is a
deterministic context-free pure function: no two honest peers can disagree on
the validity of an individual proof. Slashing is immediate, locally, per peer who
receives a proof. The only convergence required is set convergence (gossip
propagation, `O(c·Δ·log N)` under anti-eclipse) — not consensus. Critically, a
self-authenticating monotone fact needs *one honest path*, not an honest majority.
The trust-closed argument of the corpus terminates here in a bounded-time social
coordination assumption, not in a committee.

**Now (v0.2):** two architecturally distinct layers, one architecture, with a
clear division of labor. **Layer 1** is the consensus-free G-Set CRDT of fault
proofs. Anyone who receives a proof slashes locally and immediately, no consensus
needed. The chain has no role in individual slashing. **Layer 2** is the PoUH
chain, redirected to a different job: its committees, per epoch, attest to the
state of the CRDT at the snapshot time — confirming which fault proofs have
propagated, and which peers have crossed rate- or pattern-based thresholds ("6
medium-severity events in 30 days"). The chain is *witness*, not *judge*. It
cannot fabricate slashes (the proofs remain self-authenticating; VERIFY rejects
forgeries regardless of what any committee signs); it can only fail to witness
true ones, in which case the next epoch's committee picks up the backlog.

**Why:** the v0.1(early) draft provided something the consensus-free design did
not: a clean mechanism for global ordering of events, which makes rate-based and
pattern-based slashing rules straightforward to specify. Pure CRDTs do not order;
ordering is what a chain is for. But the v0.1(early) draft made the chain
*load-bearing for individual slash propagation*, which reintroduces the
honest-majority assumption and weakens the manifesto reconciliation (Thesis 13's
"no central authority" lives in tension with Thesis 10's "consensus layer
assigns each attested machine one voice"). The two-layer design keeps what each
mechanism does best: the CRDT for fast consensus-free slashing under the weakest
possible assumption; the chain for ordered, finalized epoch-summaries of state
that has already been established by the CRDT.

One consequence is graceful degradation: a compromised PoUH committee for one
epoch cannot create false slashes (the proofs are still self-authenticating); it
can only delay witnessing. A pure-chain design would have suffered a more serious
failure under the same compromise. A second consequence is bootstrap softening:
the CRDT works from day one with one honest path; the PoUH chain matures as the
attested population grows past the size at which committees of 64 become
statistically defensible. The genesis bootstrap is no longer a forced choice
between "we have a chain" and "we have nothing."

On novelty: "PoUH consensus" presented alone does not engage with prior art —
Oasis Network's confidential validators, Phala's TEE-attested workers, the broader
family of VRF-selected committee BFT mechanisms. The honest novelty claim is
narrower and stronger: it is the *combination* of PoUH-attested committees as an
ordered-witness layer above a consensus-free CRDT of self-authenticating proofs.
That combination, as far as the present draft authors know, is new, and it is
substantially more defensible under expert critique than the
consensus-mechanism-alone framing.

### 6 · Manifesto edits

**Was:** sixteen numbered theses, with "we refuse to issue a token" as an absolute
and "we build by barter" implying barter was the only economy.

**Now:** seventeen theses. Thesis 7: "we build *a community* by barter" — the
community's economy, not the protocol's only one. New Thesis 9: "we build memory
into the network" — explicit endorsement of the semantic cache. Thesis 10: "the
protocol's consensus layer assigns each attested machine exactly one voice" —
explicit endorsement of PoUH, scoped to the witness layer. Thesis 15: "we refuse
to issue a token *at the protocol layer*" — allowing economies on top to use
whatever instruments they declare. New Thesis 17: "we refuse to extract rent" —
the foundation does not capture the network it built.

**Why:** the original wording, taken literally, would have made the cache and the
economic pluralism inconsistent with the manifesto itself. The edits move from
absolute refusals to layered ones — refusing at the protocol layer what we cannot
allow there, permitting on top what we cannot, and should not, forbid.

A residual tension is acknowledged and not erased: Thesis 10's consensus layer is
in literal tension with Thesis 13's refusal of central authority. RFC-0004 v0.2's
two-layer architecture reconciles them in practice — the PoUH layer is rotating,
non-load-bearing for individual slashes, and gracefully degradable — but the
reader is invited to test whether the reconciliation is sound.

---

## RFC-0004 · v0.2 → v0.3 · 2026-05-20 · A-as-bond for high-stakes acts

A focused amendment closing — partially, and honestly — a residual that the
v0.2 "strategically inert" argument left implicit.

### Was (v0.2)

The §Tenure section presented a trilemma against farmed tenure: a Sybil with
mature `T` faces three branches — never used (zero influence), used honestly
(it *is* the work the network wanted), used to collude (attributable fault
zeroes the tenure). The argument was claimed complete ("no fourth branch").

It was not complete. It implicitly assumed the one-shot payoff of a collusive
act is less than the slash value `S_NCS`. For high-stakes acts — Tier 3
verification on a multi-million-dollar query, a fork-recovery vote, a
perennial high-value cache write — the single-act payoff can exceed `S_NCS`.
A defector with mature tenure could execute such an act once, accept the
slash, and come out ahead. The branch existed; it was just not named.

A related clarification on the framing in earlier draft cycles: the
"patient adversary farming 10,000 Sybils for years" scenario was imprecise.
Under RFC-0005's PoUH, each Sybil requires real attested hardware — 10,000
attested identities cost the floor RFC-0005 tabulates (~$7M/year in
cloud-TEE rental, more for owned hardware). PoUH already constrains the
mass-Sybil patient farming variant; what it does not constrain is *one
legitimately-attested identity that defects once decisively*. That is the
residual that this amendment addresses.

### Now (v0.3)

A new mechanism: **A-as-bond for high-stakes acts**. Acts whose plausible
one-shot payoff could exceed `S_NCS` require the actor to lock a quantity of
`A` (active reputation) as a temporary bond, slashed if the act is later
determined malicious. The bond is released after a dispute window if no fault
proof emerges.

The mechanism exploits the (A, T, κ) split already in the spine. `A` decays
fast (δ_A, minute-scale) and **cannot be accumulated in advance**. Tenure can
be farmed slowly under κ; active reputation cannot. A defector with mature
`T` but neglected `A` therefore cannot bond enough for a high-stakes act.
The only path to a bond large enough is sustained recent honest participation
— which is the behavior the network wanted. The "patient farm + one-shot
cash-in" path is closed.

RFC-0004 v0.3 enumerates the act classes that require bonds (Tier 3
verification committee membership, L2 PoUH committee role, perennial
high-value cache writes, fork-recovery votes, cross-economy settlement
intermediation) and specifies the bond formula per class.

The trilemma becomes a quadrilemma — explicit fourth branch added: *used to
collude at high stakes → requires an A-bond ≥ the payoff; the bond can only
be assembled by sustained recent honest participation; therefore the cost of
attempting the act is at least the cost of being honest first*. The
inversion-by-amortisation argument that runs through the corpus reapplies at
this finer grain.

### Why

The earlier framing implied that the patient-adversary problem was mass
quantity. It is not. PoUH (RFC-0005) bounds the mass case at hardware-cost
floors that meaningfully deter all but state-actor budgets. The real
residual was always single-identity defection on payoffs higher than the
standard slash value. A-as-bond is the structural mechanism that closes this
because — by spine construction — `A` is the resource that cannot be bought
cheaply in advance. The defence reuses the existing (A, T, κ) primitive
rather than adding a new one, which is the small-surface-area discipline the
manifesto asks for.

What remains, acknowledged: state-actor-scale adversaries with both the
budget for genuine sustained participation and a specific high-value target
whose payoff exceeds bondable `A`. This is the irreducible residual — the
same shape as every "trust-closed" claim in the corpus. The amendment does
not claim to close it. It claims to narrow it.

---

## RFC-0009 v0.1 → v0.2 + RFC-0002 v0.1 → v0.2 · 2026-05-20 · Y_canon caching + embedding governance

Joint amendment addressing three open BLOCK items from the post-0008/0009
criticality re-check: B-NEW-1 (producer compute economics), B-NEW-2 (cache
content Y_canon vs Y_fast), and B5 (embedding model upgrade governance from
the original pre-implementation review).

### B-NEW-1 + B-NEW-2 — RFC-0009 v0.2 + RFC-0002 v0.2

**Was:** RFC-0009 v0.1 said "the producer serves whatever it prefers … and
additionally commits to the canonical output." This was ambiguous about
what the user receives and what gets cached. Implicit consequence:
producers paying 2× compute on every served query (not the +6%
network-overhead figure RFC-0009 v0.1 cited at the producer level), and a
silent question of whether the cache stores `Y_fast` (user-visible but
unverifiable) or `Y_canon` (verifiable but divergent from what the user
got).

**Now:** the canonical recipe is the unified output. What the producer
commits to, what the cache stores, what the user receives — all `Y_canon`.
The producer's internal serving path is an implementation choice: if it
produces `Y_canon`-equivalent outputs natively, the 2× tax is zero; if
not, the tax applies but is the producer's economic decision.

The cache property that every retrieval returns bit-identical bytes for
the same entry is a *strength*, not a bug. It makes Tier 2 audits of cache
hits trivial — under the integer-canonical recipe, σ is essentially zero
and any discrepancy is a slash. And it forces network-wide convergence
on the canonical recipe as the production serving path, which is the
intended outcome of the whole verifiable-inference layer.

**Why:** removes the silent ambiguity about user/cache divergence; makes
the producer economy explicit and binary (zero tax or full tax, no
hidden middle); gives Tier 2 audits a clean property to rely on. The
v0.1 framing was technically defensible but operationally underspecified.

### B5 — RFC-0002 v0.2 §Governance of the canonical embedding model

**Was:** RFC-0002 v0.1 named the canonical embedding model as "the
single most consequential governance decision the cache layer makes"
but did not specify the process. The flagship governance object of the
protocol's only true coordination point was left as a hand-wave.

**Now:** a new §Governance of the canonical embedding model specifies
the proposal/review/decision/transition flow:

- **Proposal**: minimum-content requirements (model hash per RFC-0008,
  perplexity comparison on a multi-language multi-domain benchmark,
  vulnerability analysis, licensing analysis, re-embedding compute
  estimate).
- **Review**: minimum eight-week public comment period (vs the standard
  one-to-three for ordinary amendments), written assessments from
  verification/identity/cache subsystem maintainers.
- **Decision**: BDFL during v0.x; two-thirds TSC majority post-v1.0
  (higher than the simple-majority for routine matters).
- **Transition**: six-to-twelve month coordinated dual-running, with
  background re-embedding incentivised by NCS.
- **Right to fork** throughout — the protocol's permissionless property
  applied to the most consequential governance act.

**Why:** the canonical embedding model is a single point of
coordination that the protocol genuinely depends on. The governance
process does not dissolve that dependency — it bounds the *abuse* of
it. The amendment makes the process explicit and public, so it cannot
be smuggled into by anyone, BDFL included. This is also the first time
the corpus distinguishes between "ordinary amendment" and "elevated
amendment" governance — a template for any future protocol-coordination
decisions of similar scope.

### What this does not close

Four BLOCK items from the pre-implementation criticality review remain
open after this amendment: B2 (bond accounting model), B4 (bootstrap
sequence), B6 (fork-recovery bond circularity), and the cross-RFC
nits/H-NEW items from the post-0008/0009 check. They are addressed in
Amendments B, C, and D following this one.

---

## RFC-0008 + RFC-0009 · added · 2026-05-20 · Technical references for implementation

Two new RFCs added together, in service of the same goal: close the
pre-implementation criticality gaps that would otherwise force the
reference-client engineers to guess on dozens of byte-level choices and one
genuinely hard numerical-recipe choice.

These are not design RFCs in the same sense as 0001–0006. They are
prescriptive engineering references — the boring glue that the design RFCs
presupposed but never wrote down. We add them now, before code begins, so
that "the spec" and "the implementation" can speak the same language.

### What was missing

A pre-implementation cold-read of the v0.3 corpus identified six BLOCK-level
gaps that would have stalled coding within weeks: an undefined canonical
numerical recipe (RFC-0003 named it as a leverage, did not specify it); an
unspecified TEE attestation wrapper format across 6 vendor families; an
absent bootstrap sequence; an unspecified embedding-model versioning
mechanism; an under-specified bond accounting model; and the fork-recovery
bond circularity. Several GAP-level issues were equally blocking in
aggregate: undefined PRF, undefined hash function, undefined signature
scheme, undefined wire encoding, undefined NCS representation, undefined
test-vector format.

The design RFCs left these underspecified for honest reasons — they are not
*design* questions, they are *engineering* questions, and writing a
prescriptive engineering reference inside a design RFC dilutes both. But
without them, the implementation cannot start.

### RFC-0008 — Wire Formats, Crypto Primitives, Reference Constants (Reference v0.1)

A living-standard technical reference that pins, in one place:

- **CBOR** ([RFC 8949](https://www.rfc-editor.org/rfc/rfc8949)) with
  deterministic encoding for protocol objects; **JSON** for human-facing.
- **BLAKE3** for protocol-internal hashing (Merkle trees, content
  addressing); **SHA-256/384** preserved for TEE-attestation compatibility.
- **Ed25519** for peer signatures; **ECDSA-P256** preserved for TEE;
  **BLS12-381** for L2 PoUH committee signature aggregation.
- **BLAKE3.derive_key** as the PRF for beacon-bound selection, with a
  reserved domain-separation table.
- **ECVRF-EDWARDS25519-SHA512-TAI** ([RFC 9381](https://www.rfc-editor.org/rfc/rfc9381))
  for VRF committee selection; **drand** (League of Entropy) as external
  entropy beacon.
- **u64 micro-NCS** (μNCS, 10⁻⁶ NCS) for all ledger arithmetic; floats
  forbidden in protocol-level objects.
- **Embedding model pinning mechanism** — BGE-M3 weights+tokenizer hashed
  with BLAKE3, exact checkpoint hash filled when the reference client
  ships.
- **A consolidated reference-constants table** of every parameter scattered
  across 0001–0006, with explicit Pinned vs Calibratable markers.
- **Test-vector framework** — a sibling repo, structure declared here,
  populated by the reference client.

Status is **Reference (living standard)** rather than **Draft** because the
document's relationship to argument is different — once a primitive is
pinned, it changes only via a coordinated breaking event, not via the
"written to be argued with" cadence of the design RFCs.

### RFC-0009 — Canonical Numerics for Verifiable Inference (Draft v0.1 early)

The hardest deliverable in the corpus, made explicit. The recipe RFC-0003
named as a leverage but did not specify, now specified — partially, and we
say so plainly.

**Default recipe**: INT8 weight quantization (GPTQ per-channel symmetric,
group 128) with integer-domain activations and pinned reduction order on the
audited path. Integer arithmetic is associative; two honest implementations
on different hardware produce bit-identical outputs. The `σ` honest-noise
floor under this recipe is essentially zero, which makes the e-process test
of RFC-0003 §Tier 2 trivial for any non-zero quantization fraud.

**Fallback**: FP16 with constrained kernel selection and deterministic
reduction order for models where INT8 quantization is not viable (some QAT
models, exotic architectures). σ is small (10⁻⁵ to 10⁻⁴) but not zero;
the e-process still works, with worse constants.

**Verifiability tax** quantified: the audited path is 2–4× slower than the
producer's fast-path serving. At p=3% sampling, this is +6% network compute
overhead. Borne by producers, not consumers.

**Reference kernel library** specified as `ants-canonical-kernels` (Rust,
hand-tuned SIMD per platform). Skeleton given; implementation is real work
— estimated 6–12 months for the CPU canonical path alone, with GPU
acceleration following per-platform. Until it exists, "canonical numerics"
is words; we say so in the RFC.

Marked **Draft v0.1 (early)** because the recipe will be revised when the
reference kernel library exists and b2 measures its honest-noise floor.
Substantial open questions are listed honestly (MoE routing, speculative
decoding, KV-cache audits, Apple Silicon achievability, closed-weight
models, verifiability-tax economics).

### Why both, why now

Implementation cannot start with byte-level ambiguity, and it cannot start
without a canonical numerics specification. Adding these now means: when the
first engineer opens an editor to write the reference client, every
primitive choice is pinned and the audited-path recipe is at least
provisionally specified. Six months of "we'll figure it out as we go"
become six months of writing code against a spec — different problems,
different costs, much better outcome.

The two RFCs together close BLOCK B1 (canonical numerics) and B3
(attestation wrapper format choices, via §1.1 CBOR + §3.4 key hierarchy),
and all the GAP items from the criticality review (PRF, hash, signature,
encoding, NCS encoding, test-vector format). The remaining BLOCK items —
B2 bond accounting model, B4 bootstrap sequence, B5 embedding model
versioning governance, B6 fork-recovery bond circularity — are smaller
and addressed via amendments to the existing design RFCs.

A fresh corpus-wide criticality re-check is the natural next step after
these land — to surface incompatibilities that the new primitives may
have introduced between what the design RFCs *assume* and what the
references *actually specify*.

---

## RFC-0008 · v0.1 → v0.2 · 2026-05-20 · Context strings, drand failover, BLS/Ed25519 transition, constants

A focused amendment closing the BLOCK and HARD items from the post-0008/0009
criticality re-check that this document is responsible for.

### B-NEW-3 — Logit-trace Merkle context strings

**Was:** §4.1's reserved-context table covered fault proofs
(`ants-v1-fault-merkle`) but not the commit-at-send Merkle that RFC-0003
§commit-at-send specifies. An implementer of RFC-0003 had `leaf_i =
H(dist_i ‖ i)` from the spec but no derive_key context to plug into BLAKE3.

**Now:** §4.1 adds `ants-v1-logit-trace-leaf` and `ants-v1-logit-trace-root`
contexts, plus `ants-v1-cache-entry-hash` for `Y_canon` content addressing
in RFC-0002 v0.2, plus `ants-v1-vrf-seed-degraded` for the fallback below.

### B-NEW-4 — Drand failover

**Was:** §4.2 bound the L2 VRF seed to drand. Real drand outages would
stall the chain entirely.

**Now:** new §4.3 specifies a degraded-seed fallback. 30-second timeout to
fetch the drand round; on timeout the proposer constructs the seed from
`BLAKE3.derive_key("ants-v1-vrf-seed-degraded", prev_block_hash ‖ height)`
alone and sets `degraded_seed: true` in the block header. Blocks finalise
normally; the degraded-seed flag is visible in chain history forever.

**Honest residual:** during outages, an attacker with 1/3 of the previous
committee can grind toward favourable next-committees. Security regression
bounded by outage duration. The fallback prevents stall, not all attacks —
the honest framing of what we can and cannot promise.

### B-NEW-5 — BLS/Ed25519 transition at bootstrap K

**Was:** §3.3 mandated BLS12-381 aggregation for all PoUH committee
signatures regardless of K. At bootstrap K=5–16 this is wasteful: BLS
per-verify is ~10ms vs ~0.05ms for Ed25519, and aggregation savings are
negligible for small K.

**Now:** §3.3 transitions based on K. At **K ≤ 16** committee members sign
individually with Ed25519 (block header carries K Ed25519 sigs + member
IDs). At **K > 16** BLS aggregate signatures apply as before. The K=16
threshold is calibrated for ≥3× savings at crossover and is itself
calibratable (`BLS_TRANSITION_K` in §7).

### H-NEW-1 — μ_0 recipe-dependent

**Was:** §7 had a single `MU_0_HONEST_NOISE_FLOOR` constant marked "TBD on
b2". RFC-0009 v0.1 implied μ_0 differs by recipe (≈ 0 for INT8 canonical,
~10⁻⁵–10⁻⁴ for FP16 fallback).

**Now:** §7 splits into `MU_0_INT8_CANONICAL` (≈ 0, the q24 quantisation
floor) and `MU_0_FP16_CANONICAL` (TBD on b2). Implementers no longer
conflate the two regimes.

### Cross-RFC consistency nits

§3.1 now explicitly says Tier 3 verification committees (RFC-0003, N=3–7)
use Ed25519 individual signatures, not BLS. §7 adds: `CHOKE_LOOP_INTERVAL`
(10s from RFC-0001, previously missing); bond formula multipliers
(`POUH_BOND_C_COMMITTEE`, `BOND_RISK_MULT_CACHE_WRITE`,
`BOND_RISK_MULT_SETTLEMENT`, all TBD on b2 — referenced by RFC-0004 v0.3
§Bonds); `BLS_TRANSITION_K` (16); `DRAND_TIMEOUT` (30s).
`BOND_DISPUTE_WINDOW` is reaffirmed at 7 days (one week) for consistency
with RFC-0004 v0.3's "default starting point: one week" phrasing.

### What this does not close

Three BLOCK items remain open after this amendment: B2 (bond accounting
model — how A is measured, frozen, decayed during bond hold), B4
(bootstrap sequence), B6 (fork-recovery bond circularity). They are
addressed in Amendments C and D following this one.

---
