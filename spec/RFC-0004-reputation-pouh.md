# RFC-0004 — Reputation: Two Layers, One Architecture

**Status:** Draft · v0.6
**Topic:** A reputation system in two layers — a consensus-free CRDT of self-authenticating fault proofs for individual slashing, and a small slow Proof-of-Unique-Hardware (PoUH) chain as ordered witness for pattern-based rules. The chain confirms what has happened. It does not create what happens. v0.3 added **A-as-bond for high-stakes acts**, v0.4 specified the **bond accounting model** and the fork-recovery bond formula. v0.5 closes four BLOCK items surfaced by the multi-persona external review: **G-Set pruning** with late-joiner protocol (N1), **partition recovery** with equivocation slashing and Σ-T fork choice (N2), **selective disclosure of receipts** as a first-class mechanism (N6, fixing previously-broken "RFC-0003-b" cross-references), and a **race-safe bond admission protocol** that does not require synchronised clocks.
**Audience:** You, if you have read enough blockchain whitepapers to be skeptical of one more — and willing to read another.
**Depends on:** [RFC-0003](./RFC-0003-verification.md), [RFC-0005](./RFC-0005-identity.md)

---

## Why this RFC has two layers, not one

The community-layer ledger (RFC-0001) lives inside each peer pair. It does not
need global consensus — the ledger is local. The semantic cache (RFC-0002) lives
across many peers as a distributed key-value store. It does not need global
consensus — peers can hold slightly different views and resolve queries
individually.

But *some* facts about peer behaviour must be global. When peer X is caught
fabricating output, the network must agree this happened — across all peers,
durably, even after the original verifier is offline. And when X crosses a
*pattern* threshold ("six medium-severity events in thirty days"), the protocol
must apply the matching slash uniformly, not differently for each peer who has
heard about it.

These two requirements have different shapes. The first — "X is slashed because
this proof exists" — is a *monotone, self-authenticating fact*. The second — "X
crossed the rate threshold in this window" — is an *ordered, aggregated
judgment*. Trying to do both with the same mechanism produces either a chain
that is load-bearing for every individual slash (and inherits an honest-majority
assumption it should not need) or a CRDT that cannot order events (and so
cannot specify rate-based rules without hacks).

This document specifies two layers, with a clear division of labor:

- **Layer 1** is a **consensus-free CRDT** of self-authenticating fault proofs.
  Individual slashing happens here, immediately, no committee needed. Each
  honest peer who receives a proof applies the slash *locally* on receipt.
- **Layer 2** is a small slow **Proof-of-Unique-Hardware (PoUH) chain** whose
  finalized blocks attest, per epoch, to the state of the CRDT — and apply
  rate- and pattern-based rules that need a globally-ordered tally.

> **The chain is witness, not judge.** It cannot fabricate a slash (the proofs
> are self-authenticating; `VERIFY` rejects forgeries). It can only fail to
> witness a real one, in which case the next epoch's committee picks up the
> backlog.

This framing matters: chain compromise degrades gracefully to CRDT-only, not
catastrophically.

---

## What is global, what is not

| Object | Where it lives | Why |
|--------|----------------|-----|
| Local economic balance (NCS per pair) | RFC-0001 local ledger | bilateral, no consensus needed |
| Cache entries | RFC-0002 DHT | peers can hold different views |
| **Fault proofs (equivocation, attributable misbehaviour)** | **Layer 1 CRDT (this RFC)** | **monotone, self-authenticating** |
| **Per-peer tenure / reputation history** | **Computed locally from L1 + receipts** | **derived, not stored** |
| **Epoch summaries (which patterns were crossed)** | **Layer 2 chain (this RFC)** | **needs global ordering** |
| **Genesis trustee set & schedule** | **Layer 2 chain (genesis block)** | **must be witnessable from cold start** |
| Hardware attestations | Per-peer, presented on demand | RFC-0005 |

The principle: **the global layer carries only what cannot be carried locally.**

---

## Layer 1 — the consensus-free fault G-Set

A fault proof `π` is a self-contained object: two conflicting signed
messages, a signed claim that does not open consistently against a Merkle root,
a signed audit verdict on an attested-fraud event from RFC-0003. The validity
check is a **deterministic, context-free, pure function** of `π` and the
subject's public key (committed at the genesis trustee level — see below):

```
VERIFY(π, pubkey)  →  { valid, invalid }
```

No two honest peers running VERIFY on the same `π` can disagree. The only
uncertainty in the system is **which proofs each peer has received yet** — set
convergence, not value agreement.

The propagation rule is therefore a **grow-only set CRDT** (a G-Set, in CRDT
literature). A peer that receives a new `π`:
1. Runs `VERIFY(π)`. If invalid, drops it and may rate-limit the sender.
2. If valid, applies the slash locally and immediately.
3. Forwards `π` to its gossip peers.

Adversary powers against Layer 1 are limited by construction:

```
W1  withhold / eclipse    keep a valid proof from reaching an honest victim
W2  propagation delay     the proof spreads, but not yet everywhere
W3  invalid-proof spam    flood VERIFY with garbage to exhaust it
```

The adversary **cannot inject false slashes** (VERIFY filters them) and
**cannot un-slash** (the set is append-only). The only security parameter is
the **propagation window**.

### The propagation bound

Epidemic gossip over `N` honest peers with fanout `f` and round interval `Δ`:

```
T_prop  ≈  c · Δ · log N      (w.h.p.)
```

After `T_prop`, a slashed identity is globally dead, permanently. The bound
holds **conditional on an anti-eclipse / honest-subgraph-non-partition
assumption** (the honest subgraph contains at least one path from detector to
victim within the window). Without that assumption, the bound is vacuous.

### The structural win

This is the asymmetry that makes Layer 1 work at all under such weak
assumptions:

> **A monotone, self-authenticating fact needs one honest path, ever — not an
> honest majority.**

Consensus needs an honest majority. Epidemic dissemination of a self-verifying
monotone fact succeeds if there exists *any* honest path from detector to
victim within `T_prop`. To suppress a proof, the adversary must achieve a
**full vertex cut** of the honest subgraph around the victim — not merely
out-number it. This is an exponentially weaker assumption than honest majority,
and it is what makes a global reputation layer possible without consensus.

### The binding cross-layer constraint

The window that matters operationally is not "until all `N` peers know."
It is "until the peers the RFC-0003 challenge mechanism is about to *pair with
the slashed identity* know." That gives the constraint:

```
   T_prop  <  T_beacon
```

A slash must outrun the verifier-rotation cadence of the witness layer, else a
freshly-slashed peer is re-selected before the network knows. This is a real
quantified trade: faster gossip (smaller `Δ`, higher `f` → bandwidth cost)
versus slower beacon rotation (worse fraud-detection latency, which feeds
RFC-0003's `p`).

### DoS

Invalid-proof spam (W3) is bounded but not prevented. Defences:

- **VERIFY-then-relay.** A peer verifies once before forwarding. Invalid
  proofs are dropped at the boundary. The cost of verifying a *novel* candidate
  is irreducible — `O(f)` per first-relay — but bounded by per-peer fanout, not
  amplified globally.
- **Accountability for spam.** Repeatedly relaying invalid proofs is itself an
  attributable fault. The same anti-equivocation primitive that defines L1 also
  defines this: a relayer who signs forwards must do so consistently.
- **Rate limits.** Per-attested-identity rate limits on novel-proof submission
  (RFC-0005 enforces "one identity per CPU" at the cost basis).

---

### G-Set pruning and late-joiner protocol

*Added in v0.5 to close N1 from the multi-persona review.*

The G-Set described above is **append-only**, which is correct for security
(monotone under attack) but not sustainable indefinitely. At steady-state
network volume — say 10⁴ fault proofs per day — five years of operation
produces ~18M proofs every peer would otherwise retain to compute `VERIFY`
and answer "is X slashed?" queries.

The L2 chain (§Layer 2 below) gives us the tool to bound L1 storage without
weakening the security argument. The mechanism: **L1 proofs become prunable
after L2 epoch confirmation, with the L2 Merkle root serving as the
canonical record from that point.**

Concretely:

- When the L2 committee finalises epoch summary `E_n`, it includes a Merkle
  root over all fault proofs visible at the cutoff time (§Layer 2 already
  specifies this).
- After epoch `E_(n+1)` finalises — one-epoch delay for cross-checking —
  proofs included in `E_n`'s Merkle root become **eligible for L1 pruning**.
  Individual peers may discard them from their local CRDT view to bound
  storage.
- Pruning is **peer-local**: each peer decides when to prune. A peer that
  wants to act as a long-term archive (and earn the corresponding NCS) may
  retain everything; a resource-constrained peer (a phone, a small home
  server) may aggressively prune.

#### Late-joiner protocol

A peer joining the network at time `t = 1 year` does **not** download 18M
historical proofs. Instead:

1. **Sync the L2 chain** from genesis to the current epoch. This gives the
   peer all `EpochSummary` objects with their Merkle roots over the L1
   proof sets active at each cutoff. This is the bounded-size record
   (small constant per epoch).
2. **Sync the current L1 CRDT**: only proofs from the active epoch plus
   the previous one. Small steady-state size.
3. **For any historical "is X slashed?" query**: look up `X` in the L2
   chain's index of slashed identities (every epoch summary lists subjects
   whose proofs it confirmed). Find the epoch where `X` was first slashed,
   identify the corresponding Merkle root.
4. **Obtain the specific proof on demand**: query the network for the
   proof from any peer that retains it (typically archive nodes — see
   below). Verify the proof against the Merkle root from step 3. If
   `VERIFY(π) == valid` and the Merkle inclusion proof matches the
   epoch's root, the slash is confirmed.

A late joiner can therefore always confirm whether a peer is slashed (the
L2 chain says so authoritatively), and can verify the underlying proof if
any archive node still holds it. The worst case is a slash whose
underlying proof is no longer retrievable from any peer — recoverable in
principle (the L2 Merkle root binds the proof's identity, and any honest
peer with the proof can re-introduce it), but in practice means the late
joiner refuses to interact with the subject until the proof is obtained.
This is the **safe direction of error**: a late joiner who cannot verify
defaults to treating the subject as slashed (per the L2 chain), not as
honest.

#### Archive nodes and the pruning-DoS defence

Long-term retention of pruned proofs is an emergent role with a
structural defence against the obvious attack. The defence has three
layers, added in v0.6 to close the pruning-DoS vector raised by the
multi-persona Round 2 review (DS persona).

**The attack.** A safe-direction-of-error in §Late-joiner protocol
treats subjects whose proofs cannot be retrieved as slashed by default.
An adversary controlling enough of the archive layer can therefore
withhold *valid* proofs and force late joiners to refuse interaction
with anyone shown as slashed in *any* L2 Merkle root — including all
honest peers whose old, minor slashes have been long since
out-weighted by κ-bound new contribution. Without an archive-availability
guarantee, the network's onboarding becomes selectively gated by
whoever runs the archives.

**Layer 1 — minimum-redundancy target with chain monitoring.** Every
proof committed to an L2 Merkle root has a **target redundancy** of
`R_ARCHIVE_MIN` distinct holders (default `R_ARCHIVE_MIN = 8`,
calibratable per RFC-0008 §7). The L2 committee, as part of each epoch
summary, includes a **rarity flag** for any proof whose observed holder
count has fallen below the target. The chain becomes the canonical
record of "which historical proofs are currently under-replicated."
This requires no consensus on hold-counts (the committee reports its
*observed* count from gossip; honest peers report their possession
through routine cache participation); under-counting is the safe
direction of error and triggers more reward, not less.

**Layer 2 — scaling NCS reward by inverse redundancy.** Archive retrieval
fees scale by `R_ARCHIVE_MIN / max(1, observed_holders)`. A proof held
by `R_ARCHIVE_MIN` or more archives earns the baseline reward per
fetch; a proof held by only 2 archives earns 4× the baseline; a proof
held by only 1 archive earns 8×. The mechanism is straightforward
supply-and-demand applied to permanence: the rarer a proof becomes, the
more valuable it is to retain. A would-be DoS adversary who has
convinced most archives to drop a proof is *paying* the remaining honest
archives to keep it.

**Layer 3 — passive late-joiner archiving.** A late joiner that
successfully fetches and verifies a historical proof (via §Late-joiner
protocol step 4) **retains it by default** for at least one epoch
beyond verification, and can choose to retain it indefinitely. New
peers thus passively contribute to archive redundancy without taking on
the dedicated archive-node role. The retention default is opt-out, not
opt-in: only resource-constrained peers (phones, IoT-class) explicitly
disable it.

**Honest residual.** The three layers raise the cost of pruning-DoS
from "drop one peer from archive duty" to "convince a quorum of archives
+ outpace the reward-scaling + suppress passive new-joiner archiving."
That is qualitatively the same posture the rest of the corpus takes
toward DoS — expensive, not impossible. A coordinated state-actor-scale
adversary remains capable of impairing access to specific historical
proofs, which is then handled the same way every other state-actor
residual is: the credible-fork-threat at the bottom of the stack.

The protocol does not require any specific peer to be a dedicated
archive node. At maturity, a handful of well-resourced peers
(commercial operators, foundations, academic institutions) are
expected to choose the role for reputation or institutional reasons —
similar to Bitcoin full-history archive nodes today. The minimum-redundancy
target and the reward scaling create the floor below which the network
treats archive availability as a problem to fix, not a feature it
tolerates.

If at some point in the future *no* peer retains a specific historical
proof, that proof becomes irrecoverable. The L2 chain still attests
that the slash happened (via the Merkle root), but the underlying
evidence is lost. With the three layers above, the conditions for this
are: the rarity flag was raised, the reward scaling failed to attract
any peer to retain the proof, and no new joiner happened to fetch it
during the rarity window. Under nominal network health none of these
should hold for any specific proof; under network collapse, archive
availability is the least of the worries.

#### What this preserves

The **one-honest-path property** of §The structural win is preserved at
the active-epoch level (recent proofs propagate through L1 as before).
For pruned proofs, the property becomes
**any-archive-node-honest-path** — different in mechanism but equivalent
in security guarantee, because the proofs remain self-authenticating and
the L2 chain commits to their existence. Pruning does not weaken
`VERIFY`; it just moves where the proof lives.

---

## Layer 2 — the PoUH chain as ordered witness

Layer 1 handles individual slashing. It does not handle *patterns* — rules of
the form "if peer X has had six medium-severity events in the last thirty days,
escalate to hard slash." These rules need a global ordered tally of events
across a window, which is what a chain is for and a CRDT is not.

The chain is small, slow, and **deliberately unambitious in scope**: it does
not handle any individual slash, only the ordered confirmation of what Layer 1
has already established.

### Consensus mechanism: PoUH

Every peer in the attested population (RFC-0005) is a member of the chain's
validator pool — equally, one CPU one voice. For each new block:

1. A **Verifiable Random Function (VRF)** seeded by the previous block hash
   and a beacon value derived from external entropy selects a **committee of
   `K = 64`** attested peers from the global attested population.
2. The committee proposer (deterministic within the committee) assembles a
   candidate block from the mempool.
3. Each committee member verifies the block's contents and signs an
   attestation.
4. **2/3 of signatures** (rounded up) finalize the block.
5. The block is gossiped to the rest of the network; any peer can verify the
   committee signatures.

Block time target: **30 seconds**. Slow on purpose — there is no urgency, and
slow makes adversarial conditions less profitable. A block that fails to reach
2/3 within a timeout (120s) is skipped; the next committee proceeds with the
same mempool.

Finality is final: once 2/3 of a committee signs, the block stays.

### What gets recorded — the epoch summary

The chain does not record individual slashes (that is L1's job). It records
**epoch summaries** — small structured objects produced periodically by
committees that have observed the CRDT state:

```
EpochSummary {
  epoch:                 u64
  cutoff_time:           timestamp        // CRDT state observed at this T
  confirmed_proofs:      MerkleRoot       // root over Layer-1 proofs visible at cutoff
  pattern_findings: [
    { peer_id, rule_id, window, count, severity }
  ]
  validator_attestations:  [ k-of-n signatures ]
}
```

The committee's job at each epoch:
1. Observe the gossiped CRDT state at the cutoff timestamp.
2. Apply the protocol-defined pattern-detection rules (see *Slash mechanics*
   below) against the visible L1 proofs.
3. Produce an `EpochSummary` containing the proofs observed and the patterns
   crossed.
4. Sign and finalize.

The committee **does not create slashes**. It confirms what Layer 1 has already
established, and applies *rules* that need ordered counting.

### Why the committee cannot fabricate

`VERIFY(π)` is deterministic and pure. A committee that publishes a summary
including a "fault proof" that does not actually verify is trivially detectable
by any peer running VERIFY independently. Fabrication is therefore not a
failure mode: the committee can only fail to *include* real proofs (suppression)
or fail to *detect* real patterns (omission). In both cases, the next epoch's
committee picks up the missed evidence.

This is the load-bearing property. **The chain's compromise scenario is
degradation, not corruption.**

### Graceful degradation

A compromised committee for one epoch — say 1/3 of the committee is
adversary-controlled — can:
- Prevent finalization (block doesn't reach 2/3 — recovered by the next epoch).
- Suppress one specific pattern finding (next epoch's independently-selected
  committee picks it up).
- Re-order events within the epoch (cosmetic; the underlying CRDT proofs are
  uncompromised).

A compromised committee **cannot**:
- Fabricate false fault proofs (VERIFY filters them).
- Reverse a slash that has happened (Layer 1 is append-only).
- Take a slashed identity out of slash (proofs persist in the CRDT regardless
  of chain state).

A *sustained* corruption of the committee (over many epochs, requiring 67%+
control of the global attested peer population) would still not fabricate
slashes — it would simply slow down or stop the chain. Honest peers continue
to operate Layer 1 throughout. The network does not collapse; it operates in a
"witness-degraded" mode until honest peers regain committee majority.

### Bootstrap softening

The v0.1(early) draft of this RFC had an explicit bootstrap problem: when the
network has only a few attested peers, K=64 committees are statistically
infeasible, and the chain has to start with "the founders' attested peers as
bootstrap committee" — concentrating early authority.

Under the v0.2 two-layer architecture this softens substantially:

- **Layer 1 works from day one with one honest path** (the asymmetry above).
  Individual slashing is operational from the moment the first honest peer is
  online.
- **Layer 2 matures gradually**: bootstrap committees of small `K` (e.g., 5 →
  16 → 32 → 64) as the attested population grows. Mis-witnessing during the
  early period only delays pattern-based rules; it cannot fabricate slashes.

The genesis trustee set (per RFC-0001 v0.2 / RFC-0007) seeds the bootstrap
committee; its decaying authority sunsets on the same `δ` curve described in
*Fork-recovery* below.

---

### Partition recovery

*Added in v0.5 to close N2 from the multi-persona review.*

The Layer 2 chain uses 2/3 finality per epoch. Under network partition this
creates a real risk that two partition halves *both* achieve 2/3 finality
on different blocks — producing two finalised chains that cannot reconcile
by "longest chain wins" logic (the problem that has broken Solana
repeatedly). The protocol specifies three composable mechanisms.

#### Equivocation slashing

A committee member that signs blocks `B_A` and `B_B` at the same height in
two different forks is committing **attributable equivocation**. The two
signatures are themselves the proof: anyone holding both signed messages
computes a self-authenticating fault proof and propagates it through L1.

The penalty is severe — terminal-slash per §Slash mechanics: the
equivocating validator's identity is permanently flagged, its tenure
zeroed, and its bond on the relevant committee role lost. With committee
size `K = 64`, achieving 2/3 finality on two conflicting blocks requires
**at least 43 equivocating validators simultaneously** — a coordinated
cost that vastly exceeds the value of any plausible attack a partition
could enable.

The deterrent is the design's primary defence. Equivocation is so
expensive that no rational validator commits it. Partitions resolved by
*"only one side achieves 2/3 because the other side's validators refuse
to sign conflicting blocks"* are the **expected** outcome.

#### Fork choice by Σ T_eff

When two finalised chains nonetheless emerge — through coordinated malice
or extreme partition — the protocol's fork choice rule for reconciliation
is the **cumulative effective tenure** of the validators that signed each
chain's blocks. The fork with higher `Σ T_eff` (summed over all
validators participating in all finalised blocks of the fork, computed
against the pre-partition agreed state) wins.

This metric is deliberately chosen: `T` is the slow, hard-to-forge
component of reputation (rate-capped by `κ`), so it is a robust proxy
for "which fork has the more legitimate validator set." A fork built on
freshly-rented cloud-TEE Sybils has low effective tenure; a fork
supported by established peers has high effective tenure.

#### The saturating T → T_eff transform

*Added in v0.6 to address the long-tail centralisation concern raised
by the multi-persona Round 2 review (DS + econ personas, independently).*

A naïve fork choice on raw `Σ T` is dominated by historical peers in the
long run: under `κ` rate-capping, a 5-year peer's `T` is roughly 5× a
1-year peer's, and a 10-year peer's is roughly 10×. In a mature network,
the founder cohort would retain fork-choice authority indefinitely under
this metric — "founder weight" rebranded as tenure dominance, in literal
tension with the §Manifesto reconciliation argument and with the
δ_genesis sunset of RFC-0010.

The protocol therefore applies a **saturating transform** to `T` *for
fork-choice purposes only*:

```
T_eff(T)  =  T_CAP · (1 − exp(−T / T_CAP))
```

Properties of this transform, all intentional:

- **Linear regime for young peers.** When `T ≪ T_CAP`,
  `T_eff(T) ≈ T` (to first order). Young honest peers retain full
  fork-choice weight in proportion to their work — the cap does not
  punish recent contribution.
- **Saturating regime for tenured peers.** When `T → ∞`,
  `T_eff(T) → T_CAP`. A peer with 10× the tenure of another contributes
  at most `T_CAP` worth of fork-choice weight, not 10× the other's
  weight. The metric becomes a count of "established peers" with
  diminishing returns above the cap, not a sum of raw tenure.
- **Monotone, smooth, deterministic.** Two honest implementations
  compute the same `T_eff` from the same `T`. No threshold cliff (the
  transform is C∞), so an attacker cannot game the cap by sitting
  arbitrarily close to it.

The cap applies **only to the fork-choice metric**. `T` itself, uncapped,
continues to govern verifier-set eligibility (§Tenure), bond capacity
(§Bonds), persistence (a peer's standing for ordinary serving and
signalling), and the slow-decay floor below which a peer is treated as
newcomer. Capping `T` for those would punish exactly the long-term
honest contribution the spine is designed to reward. The fork-choice
metric is the only place where the *relative weight* of an arbitrarily
old peer needs bounding, because fork-choice is the only place where
one peer's weight relative to another determines the network's path.

`T_CAP` is **calibratable** (added to RFC-0008 §7 as
`T_FORK_CHOICE_CAP`). A reasonable starting point is the tenure
accumulable in roughly two years of sustained κ-rate-cap-bound
contribution; the precise value is a b2-class measurement once `κ` is
fixed.

The losing fork's blocks after the partition cut are **reverted** — they
become "uncle blocks" in the reconciled history. The validators of the
losing fork are not slashed for signing those blocks (they signed only
one fork's blocks; no equivocation), but their work for that fork is
discarded. Bond admissions, slashes, and pattern findings recorded only
in the losing fork are reissued through the winning fork's subsequent
epochs.

#### Social-Schelling fallback

If both forks have `Σ T_eff > θ · Σ T_eff,total` — the partition was
fundamentally balanced and both sides have legitimate majority validator
support — fork choice by `Σ T_eff` is ambiguous and the protocol
explicitly hands recovery to the social layer per §Fork-recovery and
the credible-fork-threat. The cap makes this fallback *more* likely to
trigger in practice, not less: with raw `Σ T`, a partition split
imbalanced in age (one side has the founder cohort, the other has the
recent cohort) might appear unbalanced enough for automatic
reconciliation when in fact the network is socially split. `Σ T_eff`
correctly reports that case as balanced and hands it off.

This is the same social-Schelling floor the architecture has at every
layer. It is the irreducible coordination assumption. Naming it
explicitly here means a balanced-partition recovery is not undefined
behaviour: it is the explicit invocation of the well-specified
fork-recovery process, with all its sub-problems and protocol
requirements.

#### What this preserves

The "chain is witness, not judge" property of §Layer 2 is preserved.
Partition recovery does not require the chain to *create* slashes — the
equivocation proofs are self-authenticating L1 fault proofs that exist
regardless of which fork wins. The chain's role in partition recovery is
to provide `Σ T_eff` accounting and the canonical history once
reconciliation happens. The L1 layer continues to operate on both sides
during the partition, accumulating proofs that will be reconciled when
the partition heals.

---

## Tenure: the (A, T, κ) spine

Reputation in ANTS is two-component, by design. The naive single-scalar
"reputation" tries to be (a) responsive to recent behaviour (choke/unchoke), (b)
durable enough for verifier eligibility, and (c) hard to acquire fast enough to
matter for Sybil resistance — three pressures that **cannot all be satisfied by
the same number on the same time-scale**.

```
   Reputation  =  ( A, T )
   A   "active"   fast-decay δ_A     serving priority + audit-window bound + genesis retirement
   T   "tenure"   slow-decay δ_T     verifier eligibility + stable fork-succession ranking
                                     acquired under per-identity rate cap κ

   Eligibility for the verifier set  ← gated by T   (slow, stable)
   Weight in any single audit / serving round ← governed by A (fast, volatile)
```

`κ` is the new structural primitive. It caps how fast tenure can accrue *per
attested identity, per unit time*, regardless of how much work an identity
performs. Spreading work across many Sybils does not accelerate any single
identity's tenure. This is what makes **time the resource that cannot be
bought**: the manifesto's "one CPU one identity" delivered not by hardware-only
(which is rentable at scale via cloud TEEs) and not by stake (which is
forbidden), but by a protocol-level cap that no amount of money or compute can
shortcut.

### How tenure interacts with Layer 1

T is **negatively** zeroed by self-authenticating fault — a confirmed
fabrication, equivocation, or cache-poisoning proof against an identity zeroes
its `T`. The zero is applied at L1 propagation time, immediately, locally.

T is **positively** never stored globally. Each peer computes its own claimed T
on demand by presenting a bag of counterparty-countersigned receipts (from
served work, audits performed, cache hosting credit). Anyone can recompute the
same `T_X` from the same receipt bag. The κ-clip is applied inside the
function: per-bin accrual ≤ `κ · (bin width)`, regardless of how many receipts
are presented for that bin.

### Why fake tenure is strategically inert

A Sybil ring can countersign each other's receipts to farm tenure that does not
correspond to real work. Layer 1 cannot detect this at accrual time. But the
farmed tenure is **strategically inert**:

- *Never used* → zero influence (Layer 2 weight in a round is governed by `A`,
  not `T`; a Sybil with farmed `T` but no recent real work has no `A`).
- *Used honestly* → it *is* the useful work the network wanted in the first
  place.
- *Used to collude* → must produce RFC-0003 commits; collusion generates an
  attributable fault proof; the proof propagates through Layer 1 and zeroes
  the tenure. Identity becomes worthless.

*Claimed in v0.2: there is no fourth branch.* The claim was implicitly
conditional on the act's one-shot payoff being bounded by `S_NCS`. **v0.3
acknowledges the implicit fourth branch** — collusion on an act whose
single-shot payoff exceeds `S_NCS` — and closes it via the §Bonds
mechanism below: such acts require an A-bond that a defector with mature
`T` but neglected `A` cannot post. With that branch named, the trilemma is
strict for the standard case and the quadrilemma is strict for the
high-stakes case. The marriage of (A, T, κ) plus L1 fault
propagation plus RFC-0003's commit-at-send is what makes the inversion work:
farmed tenure is uncatchable at accrual, but useless without being used, and
self-destructive when used. This adds *no new* security parameter — its
integrity reduces exactly to:

```
tenure integrity  ⟺  (RFC-0003 deterrence inequality)  ∧  (T_prop < T_beacon)
```

— the conjunction of two already-derived constraints. The keystone behaving as
keystone: it does not add a wall, it distributes the load onto walls already
standing.

---

## Slash mechanics

Slash *triggers* live in two places, by purpose:

| Severity | Trigger | Mechanism | Recorded by |
|----------|---------|-----------|-------------|
| Immediate | Individual fault proof verifies | L1 propagation; each honest peer applies locally on receipt | L1 CRDT |
| Soft | 1–2 fault events in 30 days | Pattern rule applied at epoch | L2 chain |
| Medium | 3–5 events in 30 days, or 1 cache poisoning | Pattern rule | L2 chain |
| Hard | 6+ events in 30 days, repeated poisoning, validator misconduct | Pattern rule | L2 chain |
| Terminal | Confirmed Sybil, deliberate consensus attack, repeated hard | Pattern rule, irreversible | L2 chain |

The severity classification, the window, and the consequences are
protocol-defined parameters (in this RFC's tunable constants table —
*deliberately omitted from v0.2 pending testnet calibration*). The
*application* of those parameters is what Layer 2 committees do at each
epoch: they count L1 events visible at the cutoff and emit pattern findings
when thresholds are crossed.

Pattern findings can also be **disputed**. A slashed peer who believes the
pattern finding is wrong (events counted twice, events whose underlying L1
proof is invalid, window mis-applied) can initiate a dispute. The dispute is
heard by a *separate* committee with longer review period, larger `K`, and
explicit access to the underlying L1 proofs. Disputes do not reverse the
L1 slash event itself; they only adjust the L2 pattern finding. This is the
correct shape — the L1 fact stands as long as VERIFY passes; the L2
*interpretation* of facts is open to argument.

---

## Bonds for high-stakes acts

*Added in v0.3.* The trilemma in §Tenure showed three branches against
farmed tenure. The implicit fourth branch — used to collude on an act whose
one-shot payoff exceeds the standard slash value — was not closed by L1
fault propagation alone. This section names that branch and closes it.

### The threat

The standard slash mechanics in §Slash work because the cost of being caught
(the value of standing destroyed, `S_NCS` from
[RFC-0001](./RFC-0001-community-economy.md) v0.3) exceeds the gain from
typical fraud — token-level inference fraud, low-value cache writes, single
cache poisoning. All bounded in single-act payoff.

Some acts are not typical. They have one-shot payoffs that can dominate
`S_NCS`:

- A Tier 3 verification committee whose query is declared at high stakes —
  medical, legal, financial — with the consumer's stakes-declaration field
  carrying real value.
- An L2 PoUH committee role where suppressing one pattern finding for one
  epoch has outsize downstream effect.
- A write of a perennial-validity cache entry in a high-value embedding
  region (legal canon, regulated-domain factual claims).
- A vote in a fork-recovery succession, where one decisive vote may tip
  which trustee set inherits authority.
- A cross-economy cache settlement intermediation
  ([RFC-0006](./RFC-0006-payment-terms.md)) where one inflated quote may
  extract value before being caught.

For these acts, the standard deterrent fails: an attacker who accepts the
slash can still come out ahead if the act's payoff exceeds `S_NCS`. The L1
fault proof zeroes their tenure; the chain confirms the slash; the identity
dies — but the value extracted is greater than the value lost.

### The mechanism

For any act enumerated above, the protocol requires the actor to **lock a
bond of `A`** (active reputation) before the act is admitted. The bond is
held for the duration of the act plus a dispute window. If a fault proof
against the act is propagated in L1 (or a pattern finding against the actor
is recorded in L2) during the bond period, the bonded `A` is *added to* the
standard slash. If the bond period elapses without a fault, the `A` is
released back to the actor's available pool.

There is no new cryptographic primitive. The slash mechanism is the same one
specified in §Slash; the bond simply lifts the slashable surface from `T`
post-hoc to `A` at the moment of high-stakes participation.

### Why this works

The mechanism does not depend on detecting *intent*. It depends on a
structural property of the spine introduced in §Tenure:

> `A` decays at δ_A on the scale of minutes. **It cannot be accumulated in
> advance.** A peer's available `A` at any moment is, by construction,
> approximately proportional to its recent honest contribution rate.

A defector with mature `T` but neglected `A` therefore *does not have enough
`A`* to post the bond required by a high-stakes act. They face a forced
choice:

- **Skip the act.** Patient farming was for nothing; no high-stakes attack
  is available to them.
- **Bond enough.** This requires having maintained recent honest
  participation at a rate sufficient to assemble the bond. *That sustained
  participation is the contribution the network wanted.* The cost of
  preparing the attack approaches the cost of being honest. If the attack
  succeeds, the bond is slashed and the prior tenure is also zeroed through
  the standard slash chain. If it fails, the actor has paid for sustained
  honest participation that benefited the network for the entire run-up.

The inversion-by-amortisation argument that runs through the corpus
reapplies at this finer grain: time is the resource that cannot be bought,
and `A` is the *fresh* expression of time. The bond closes the high-stakes
branch by demanding fresh participation that, by definition, cannot be
backdated.

### Bond formulas

Per act class, the required bond. The principle holding across all five:
bond ≥ plausible one-shot payoff, so the act is not profitable even if not
caught by the standard slash:

| Act class | Bond formula | Rationale |
|---|---|---|
| Tier 3 verification committee member | `query_stakes / N` per member | If all `N` collude maliciously on an act of value `query_stakes`, the total bonded `A` slashed equals the act's worth — EV ≤ 0 |
| L2 PoUH committee signer | `c_committee · pending_findings` per epoch | Caps the value a corrupt committee can sell by selectively suppressing pattern findings |
| Perennial high-value cache write | estimated lifetime royalty value × risk multiplier | The write earns royalties over years; the bond covers expected total payoff plus a margin |
| Fork-recovery vote | total `T` at stake **as of the pre-fault agreed state** (per §Fork-recovery sub-problem 2) | The maximum case — the architecture's fixed point depends on this not going rogue, so the bond is the most expensive. v0.4 clarifies: the bond is computed against the *pre-fault* state because the post-fault chain is by construction in dispute — the same timestamp that legitimises the fork defines the bond input |
| Cross-economy settlement intermediation | settlement value × risk multiplier | If the settlement is inflated, the inflation is recoverable from the bond |

The formulas are starting points. The constants and risk multipliers
(`c_committee`, the `× risk multiplier` factors) are explicit calibration
parameters of this RFC, expected to be revised after testnet observation —
the same b2-style discipline applied elsewhere in the corpus.

A consequence worth flagging: the bond required for fork-recovery voting
scales with the *entire network's* `T` at stake. In a mature network this
makes participating in a fork-recovery vote *very* expensive in `A` — which
is intentional. The credibility of the fork-threat depends on its voters
not having been bought; the bond ensures that any vote requires costly
fresh skin-in-the-game.

### What this does not close

A state-actor-scale adversary with both the budget for sustained genuine
honest participation *and* a specific high-value target whose payoff
exceeds the maximum bondable `A` remains a residual. This is acknowledged.
It is the same residual the manifesto's "stewarded by a foundation"
implicitly addresses with the credible-fork-threat: if a sufficiently
determined adversary commits to enough genuine work to mount one decisive
act, the ultimate defence is the community's ability to detect, fork, and
re-seed — the bounded social-coordination assumption at the architecture's
floor.

What v0.3 claims is narrowing, not closing. The residual was previously
*any defector with mature tenure can cash in once on a high-payoff act*. It
is now *only an attacker who has paid for years of genuine network
contribution AND is targeting an act whose value exceeds even what years of
contribution can bond*. That is a substantially smaller threat surface than
the pre-v0.3 framing, and it is the smallest residual the corpus has so far
demonstrated against patient defection.

---

## Bond accounting model

*Added in v0.4 to close B2 from the pre-implementation criticality review.*

§Bonds above says "lock a bond of `A`". This section specifies how that lock
is computed, admitted, frozen, decayed during the hold, released, slashed,
and composed across overlapping acts. Without these mechanics specified, the
v0.3 bond mechanism is unimplementable.

### Computing A

Active reputation `A(peer, t)` is a non-negative `u64 μNCS` at every point
in time (encoded per [RFC-0008](./RFC-0008-wire-formats.md) §6). It is the
weighted sum of the peer's verified recent contributions with exponential
decay at rate `δ_A`:

```
A(peer, t)  =  Σ over verified contributions c_i at time t_i  of
                c_i · exp( -δ_A · (t - t_i) )
```

A "verified contribution" is any of: a Tier 1 inference served, a cache hit
served, an audit completed, a cache hosting hour, a routing hop served —
each weighted by its μNCS value per [RFC-0001](./RFC-0001-community-economy.md).
Contributions are recorded as **counterparty-countersigned receipts** per
RFC-0003 §"Tenure: default-grant-and-revoke" (the receipts structure that
already supports tenure computation also supports `A` computation).

`A` is **locally recomputable**. Anyone holding the peer's bag of
counterparty-countersigned receipts can compute the same `A(peer, t)`. It
is not stored as a number anywhere; it is derived on demand. This is the
same property tenure `T` has — both `A` and `T` are deterministic
functions of the receipt bag, differing only in their decay rates `δ_A` /
`δ_T` and (for `T`) the per-identity rate cap `κ`.

### Bondable A vs total A

The peer's **bondable `A`** at time `t` is its total `A` minus all
currently-locked bonds:

```
locked_total(peer, t)  =  Σ over currently-locked bonds  of  bond.amount
bondable_A(peer, t)    =  A(peer, t)  -  locked_total(peer, t)
```

To enter a high-stakes act requiring a bond of amount `b`, the peer proves
to the act's verifier that `bondable_A(peer, t) ≥ b`. The proof is the
peer's receipt bag (or a selective-disclosure subset per
[RFC-0003](./RFC-0003-verification.md)) plus a list of the peer's currently-
locked bonds. The verifier recomputes `A`, sums the locks, and admits the
new bond if and only if there is sufficient headroom.

### Freezing the bond

When a bond of amount `b` is admitted at time `t_start` for an act with
dispute-window duration `BOND_DISPUTE_WINDOW` (RFC-0008 §7, default 7 days):

- The bond enters the peer's `locked_bonds` set as the tuple
  `(act_id, b, t_start, t_release, slash_target)` where
  `t_release = t_start + BOND_DISPUTE_WINDOW` and `slash_target` is the
  L1 fault-proof signature that would trigger the slash if filed.
- The amount `b` is **frozen at its admitted value for the duration of the
  hold** — it does not decay during the hold. The locked portion is
  conceptually in escrow, isolated from the rest of `A`.
- During the hold, `bondable_A(peer, t)` is `A(peer, t) - locked_total`. The
  peer's serving-priority weight in the choke/unchoke loop (RFC-0001) and
  eligibility weight in verifier selection use `bondable_A`, not full `A`
  — the locked portion does not count toward priority during the hold.
- All bond-admission events are signed by the admitting verifier and
  propagated to the L1 CRDT alongside fault proofs. Other peers verify the
  signature; the bond is publicly observable as locked.

### Release and slash

At time `t_release`:

- **If during `[t_start, t_release]` a fault proof was propagated in L1
  against the peer for this specific `act_id`**, the bond's `b` is added
  to the standard slash event — destroying the locked amount along with
  the rest of the slashed reputation. The bond is removed from
  `locked_bonds` after the slash event finalises (L2 epoch attestation
  per §Slash mechanics).
- **If no fault proof for this `act_id` exists at `t_release`**, the bond
  is released cleanly: removed from `locked_bonds` and returned to the
  peer's available reputation pool. The released amount is treated as if
  it had been a fresh contribution at `t_start` — i.e., it carries forward
  in the decay function with the decay clock starting at `t_start`, not at
  `t_release`. A bond that was held for the full dispute window has
  effectively been *decaying mentally* during the hold even though the
  locked value was frozen.

The mental model: the bond is a *temporary withdrawal* from `A`'s
exponentially-decaying pool, not a *fresh contribution*. Holding doesn't
gain you any decay-clock advantage. This prevents a defector from
"refreshing" their `A` by repeatedly locking and releasing bonds.

### Multi-act composition

Multiple acts in flight simultaneously each carry their own bond. The locks
are **additive**:

```
locked_total(peer, t)  =  Σ over (act_id, b, t_start, t_release, _)
                          in locked_bonds(peer)  of  b
```

A peer with `A(peer, t) = 1000` μNCS and two locked bonds of 300 μNCS each
has `bondable_A = 400` μNCS. Attempting a third act requiring a bond of
500 μNCS fails: the peer does not have enough headroom.

This is the same accounting any decent escrow system uses, applied to
reputation rather than to currency. The novelty is the source of `A`
(RFC-0001's contribution-ledger via counterparty-countersigned receipts),
not the bond mechanic itself.

### Atomicity

Bond admission is an atomic operation: either all of (verifier recomputes
`A` and headroom, the bond is added to `locked_bonds`, the peer is admitted
to the act, the admission is signed and propagated in L1) happen, or none
happen. A peer participating in three Tier 3 committees simultaneously has
three distinct bond records, each admitted at its own timestamp.

If two acts arrive for the same peer simultaneously and both require bonds
whose sum exceeds `bondable_A`, the protocol uses a **race-safe admission
protocol** (added in v0.5 to close the bond-admission race condition from
the multi-persona review) that does **not** depend on synchronised clocks
across verifiers:

1. A verifier `V` that decides to admit a bond signs a `bond_admission`
   object containing `(act_id, peer, b, V_id, current_L1_view_hash)` and
   gossips it through L1.
2. `V` then waits a **propagation delay** `δ_admission` (default 5 seconds
   for time-sensitive acts like Tier 3 verification; longer for slower
   acts) to observe whether any conflicting admission for `peer` was
   propagated within that window.
3. If a conflicting admission is observed, **deterministic tie-break**
   resolves: the admission with the smaller
   `BLAKE3.derive_key("ants-v1-bond-tiebreak", epoch_seed ‖ act_id ‖ V_id ‖ admission_round)`
   wins. The losing admission is null; the losing verifier gossips a
   `bond_admission_null` event to L1, releasing the temporary lock.
4. If no conflict is observed after `δ_admission`, the admission is final.

The tie-break function uses the current epoch's seed as input, which
prevents a peer from precomputing favourable admission timings to game
the function. The protocol's clock-sync requirement is **loose**:
verifiers must agree on the current epoch and the current `epoch_seed`,
both of which come from the L2 chain. They do **not** need to agree on
absolute wall-clock timestamps within milliseconds.

A peer whose admission loses the tie-break must choose whether to retry
the failed act later, after `A` has grown or another bond has released.

There is no "partial bond" or "borrowing against future `A`". The bond
must exist in the peer's currently-decayed `A` at admission time.

### Verifier responsibility

The verifier of a high-stakes act (per the act class — RFC-0003 §Tier 3
for triangulation committees, this RFC §Layer 2 for PoUH committees, etc.)
is responsible for performing the bond-admission check. Specifically:

1. The peer's presented receipts pass §Selective disclosure of receipts'
   integrity checks (countersignatures valid, nonces fresh, timestamps
   attributable, no double-counting, Merkle inclusion against `bag_root`).
2. Computing `A(peer, t)` from the receipts gives ≥ the required bond.
3. The peer's declared `locked_bonds` set is consistent with what L1 has
   already propagated (the verifier can cross-check against the L1 CRDT
   view).
4. The bond admission event is signed by the verifier, added to L1, and
   the act proceeds.

A verifier who admits a bond against insufficient `bondable_A` is
committing an attributable fault — the receipt math is a deterministic
pure function, and any honest re-verifier will detect the discrepancy
and propagate a fault proof against the misadmitting verifier. This
makes bond-admission verification itself self-auditing through the
standard L1 mechanism.

### A consequence the design intends

Because `A` decays at `δ_A`, and bonds are frozen at admission time, a
peer that participates heavily in high-stakes acts is naturally limited in
how many acts it can take on simultaneously. A peer with low recent
contribution has low `A` and therefore low `bondable_A` and therefore
limited high-stakes-act capacity. A peer with high recent contribution
has higher capacity. The market clears on contribution, not on stake.

This is the same property the (A, T, κ) spine was designed for, now
applied to multi-act allocation: high-stakes participation is gated by
recent honest contribution, not by accumulated capital, not by hardware
volume, not by stake. The bond mechanic is the operational expression of
the manifesto's "the network adapts to humans, not the other way around"
at this layer.

---

## Selective disclosure of receipts

*Added in v0.5 to close N6 from the multi-persona review. Previous versions
of this RFC referenced "RFC-0003-b" for this mechanism — that reference was
a working nickname that never crystallised into a separate document. The
mechanism lives here, where the (A, T, κ) spine and bond accounting also
live.*

A peer presenting its receipts for `A` or `T` computation — to a
bond-admission verifier, to a verifier-eligibility check, to a fork-time
ranking calculation — does *not* want to reveal its entire interaction
history. The receipts collectively constitute the peer's full graph of
who-served-whom-when, which is the "ledger is private" property
[RFC-0001 v0.3](./RFC-0001-community-economy.md) explicitly preserves.

The challenge: prove `A(peer, t) ≥ b` (or `T(peer, t) ≥ FLOOR`) by
revealing the **minimum** receipt subset sufficient for the proof.

### The mechanism

A peer maintains its receipt bag as a **Merkle tree** over individual
receipt leaves, ordered by timestamp. The root is committed on demand:
the peer publishes
`bag_root = BLAKE3.derive_key("ants-v1-receipt-bag-root", peer_id ‖ committed_receipts)`
whenever it participates in a high-stakes act (RFC-0008 §4.1 reserves
this context).

To prove `A ≥ b`:

1. The peer selects a subset `S` of its receipts such that
   `Σ_{r∈S} decayed_value(r, t) ≥ b`. The selection prefers **most
   recent first** (highest decayed value per unit of revealed
   information), with ties broken by largest contribution value.
2. The peer reveals `S` along with **Merkle inclusion proofs** for each
   receipt in `S` against `bag_root`.
3. The verifier verifies each receipt's countersignature (per the
   normal receipt-bag integrity rules), checks Merkle inclusion against
   `bag_root`, and recomputes `A` over `S` to confirm the bound.

The verifier accepts iff every receipt in `S` is valid, every Merkle
proof is valid, and `Σ decayed_value over S ≥ b`.

### Why this is sound

Receipts in `S` produce a **lower bound** on the peer's total `A`. The
verifier sees a partial history; the peer's true `A` may be higher
(unrevealed receipts exist). This is the safe direction of error — a
peer that *understates* its `A` cannot exceed its true `A` through
selective disclosure.

The reverse direction is **not** a risk: a peer cannot *overstate* its
`A` by revealing receipts, because every revealed receipt is verified by
countersignature. Fake receipts fail verification.

The **positive/negative asymmetry** that the architecture relies on at
the L1 level applies here at the receipt level. A peer controls what
positive evidence (receipts) it reveals, but cannot control its negative
evidence (fault proofs in L1) — those reach the verifier independently.
A peer with mature receipts and zero faults can prove `A ≥ b`
selectively; a peer with faults cannot hide them by selective disclosure.

### Compact summaries for large peers

A peer with thousands of receipts may publish a **compact summary** to
reduce per-bond-admission verification cost: a list of
`(time_bucket, total_decayed_value_in_bucket)` pairs, signed by the peer
and committed via the same Merkle root. The summary is a *hint*; the
verifier may still require selective disclosure of individual receipts
to confirm specific buckets at random.

The summary mechanism is an optimization, not a security primitive: the
peer can always be challenged to reveal underlying receipts for any
audited bucket. A peer that publishes a summary containing false
buckets and is then audited is committing equivocation against its own
signed summary — itself an attributable fault.

Bucket granularity is a calibratable parameter (see §What we have not
figured out yet).

### What this does not eliminate

The full receipt bag is still derivable by any party that observes the
peer's interactions over time. A persistent network adversary can
correlate receipts across multiple acts (each act reveals a subset; the
union over time approaches the full bag). Selective disclosure provides
**privacy per act**, not **privacy over time**. The latter would
require zero-knowledge proofs of the same statements — implementable in
principle but at substantial cost. The protocol does not currently
specify ZK selective disclosure; the door is left open in §What we have
not figured out yet for future amendment.

---

## Fork-recovery and the credible-fork-threat

The architecture's fixed point is here. The credible-fork-threat is the
deterrent that constrains both the genesis trustee set (during the bounded
[0, t_safe] window) and the long-tail of the witness committee. It is not
trustless. It is trust-minimised, transparent, and bounded.

The earlier draft cycles of this document specified five sub-problems that
fork-recovery must solve precisely without re-introducing the authority that
declares which fork is legitimate. They are retained without weakening:

1. **Legitimacy = a cryptographically attributable fault**, enumerated in
   advance — not an accusation or a vote (a vote is symmetric:
   fork-griefing). Equivocation is the gold standard. **Requirement that
   follows: the genesis trustee schedule and every grant must be public,
   signed, append-only commitments**, so over-issuance becomes attributable
   equivocation. Residual: *censorship is not attributable* — a smart malicious
   trustee censors rather than equivocates. This is the irreducible social
   weak spot.
2. **The successor is derived, not chosen.** Deterministically from the last
   agreed pre-fault state plus a pre-committed, time-phased succession rule:
   a pre-named backup-set *sequence* near `t ≈ 0` (weakening the assumption
   from "trust this set" to "trust that at least one of a public sequence is
   honest"), handing over to the top earned-tenure peers as `W_h` grows. The
   recovery's trust source self-decentralises on the same `δ/W_h` curve the
   genesis weight decays on.
3. **State portability.** A fork must carry honest earned standing forward —
   conditionally; only the part verifiable against the pre-fault agreed state
   ports, the fault proof's timestamp defines the cut. (This is the
   security-parameter reclassification that RFC-0001 v0.3 made for the local
   ledger; same principle at this layer.)
4. **Griefing symmetry.** A false fork call produces no valid fault proof, no
   Schelling convergence, and the false fork dies. The same precision that
   enables real recovery neutralises its inverse.
5. **The irreducible floor.** Even with all of the above, recovery works only
   if a majority of honest *weight* actually relocates. Nothing in the
   protocol can force this. **The protocol drives the activation energy
   toward zero** — trivially verifiable proof, deterministic successor,
   portable state — but does not eliminate the coordination assumption. The
   protocol's source declares its own non-algorithmic fallback in writing.
   That declaration is the entire difference between ANTS and the systems
   (Tor's dirauths, every PoS chain's foundation premine, Certificate
   Transparency's trusted operators) that have this trust root and pretend
   they do not.

The attributable-fault primitive does **quadruple duty** across this
architecture: (i) Layer 1 transitive verifiability of validators; (ii)
fork-recovery legitimacy; (iii) the portable-state cut at the fault-proof
timestamp; (iv) instant tenure-slashing through L1. One primitive, four jobs —
the structural economy the manifesto's "small surface area" asks for, emergent
rather than imposed.

---

## On novelty

A note we owe the reader, because the v0.1(early) draft of this document
overclaimed.

**PoUH presented alone is not genuinely new.** It is close to existing designs
that we want to engage with honestly:

- **Oasis Network** uses TEE-attested confidential validators with a
  delegated-stake selection mechanism. The TEE-attestation-of-validators
  pattern is theirs, prior to ours. The difference is that Oasis still
  selects validators by stake; PoUH selects by attested-CPU-uniqueness
  weighted equally — replacing plutocracy with hardware-cost.
- **Phala** uses TEE-attested workers for trusted computation with a
  separate consensus layer. The attestation pattern is shared; the consensus
  use is different.
- **VRF-selected committee BFT** is well-explored — Algorand, Ouroboros
  Praos, Internet Computer's NNS subnet, several others. The selection
  mechanism is standard cryptographic primitives in standard composition.

**What is actually new in this RFC is the combination**: PoUH-attested
committees as an **ordered-witness layer above a consensus-free CRDT of
self-authenticating proofs**. The CRDT-on-the-bottom / chain-on-top
architecture, with the chain explicitly cast as witness rather than judge, is
the contribution. We claim novelty for the combination, not for the
mechanism in isolation. The narrower claim is more defensible under expert
critique, and we would rather make a claim we can defend than a claim
that sounds bigger.

If a reader knows prior art that anticipates this specific combination, we
would like to hear it — open an issue tagged `rfc-0004`.

---

## Manifesto reconciliation

The corpus has accumulated a real tension we will not erase. Thesis 13: *"We
refuse central authority over the protocol."* Thesis 10: *"the protocol's
consensus layer assigns each attested machine exactly one voice."*

A consensus layer with VRF-selected committees is, in literal terms, a
*coordination layer with limited authority over which global facts get
recorded*. PoUH softens this — committees rotate, no permanent operator
exists, validator slots cannot be bought — but it does not eliminate it. The
v0.2 two-layer architecture reconciles the two theses in practice:

- **Thesis 13 is preserved at Layer 1.** No committee, no coordinator, no
  authority touches individual slashes. The CRDT works under one-honest-path,
  not under honest-majority. The consensus-free design is what makes
  RFC-0004's *foundational* layer honest to Thesis 13.
- **Thesis 10 is delivered at Layer 2.** PoUH committees produce ordered
  epoch summaries — exactly the "one voice per attested machine in the
  consensus layer" the thesis declares. The committee structure is
  rotating, transient, and non-load-bearing for individual slashes, so its
  "authority" is sharply bounded.

The reconciliation is real but partial. A reader who concludes that Thesis 10
and Thesis 13 cannot both hold literally has, in our view, an argument we
would want to read. Open an issue.

---

## What we have not figured out yet

Honest open problems, in roughly decreasing order of how much we expect them
to bite:

- **`T_prop < T_beacon` is asserted, not measured.** Real gossip latency
  under partially-adversarial topology is the number a testnet must produce,
  alongside RFC-0003's `e` and RFC-0001 v0.3's `w`.
- **Anti-eclipse needs Sybil-resistant peer selection** — which is RFC-0005,
  which itself benefits from this layer. The recursion is mitigated by the
  one-path-not-majority asymmetry (Layer 1 needs *much* less Sybil-resistance
  than a chain would), but it is not eliminated.
- **Committee size `K`** (currently 64) is conservative. A smaller `K` is
  cheaper per block but more vulnerable to a single epoch's committee bias;
  a larger `K` is slower. Calibration pending.
- **Block time** (currently 30s) interacts with `T_prop`. If `T_prop` is
  smaller than block time, every epoch summary is current; if larger, some
  L1 events lag into the next epoch. The right block time depends on the
  ratio.
- **Epoch-summary canonicalization.** Two honest committees observing
  slightly different CRDT cutoff states should produce *compatible* (not
  identical) summaries. The exact canonicalization rule — which proofs
  count as "visible at cutoff" — needs spec.
- **Recovery from network partition.** If the network splits, do we have
  one chain or two? Standard answer: longest-valid-chain wins, but Layer 1
  continues operating on both sides during the split. Reconciliation when
  the partition heals needs spec.
- **Bootstrap committee transition.** Specific `K = 5 → 16 → 32 → 64`
  schedule by population threshold is a guess, not a derivation.
- **State growth.** Even at modest event volume, the chain grows.
  Pruning policy (archive nodes vs full nodes) is undefined.
- **Vendor compromise** at the attestation-signing layer (Intel, AMD,
  NVIDIA, ARM, Apple). Mitigation: multi-vendor diversity, with chain
  weighting that downweights any single-vendor majority. This is critical
  to RFC-0005 and to the chain's long-term security.
- **Censorship at L2** is not cryptographically attributable. A committee
  that selectively omits pattern findings is detectable statistically
  (over many epochs) but not by a single proof. Open.
- **`δ_A` calibration** (v0.3). The §Bonds mechanism rests on `A` decaying fast
  enough that a defector cannot accumulate it in advance — but slow enough that
  legitimate peers do not constantly re-earn it. Probably minutes-to-hours;
  empirical, b2.
- **Bond formula multipliers** (v0.3). The starting bond formulas in §Bonds
  give the shape (bond ≥ plausible one-shot payoff); the constants and risk
  multipliers (`c_committee`, the explicit `× risk multiplier` factors) are
  expected to be revised after testnet observes real high-stakes act
  distributions.
- **Dispute window duration for bond release** (v0.3). Too short and provable
  malice may not surface in time; too long and bonds tie up productive `A`.
  Default starting point: one week. Empirical.
- **State-actor + bonded high-stakes act** (v0.3). An attacker willing to
  sustain genuine honest contribution long enough to bond a high-stakes act,
  and to pay both the bond and the prior contribution cost for a sufficiently
  valuable target, remains an irreducible residual — handed to the
  credible-fork-threat as the last line of defence.
- **Bond admission atomicity under L1 propagation latency** (v0.4). Race
  condition: verifier V1 admits a bond for peer P's act A at time `t`,
  verifier V2 simultaneously admits a different bond for act B before V1's
  admission has propagated. Both verifiers independently checked
  `bondable_A` and both saw the same headroom — leading to over-locking.
  Resolution probably: the later-timestamped admission is invalidated when
  the conflict is detected at L1 propagation, and the peer must re-acquire
  the lost slot. Exact resolution mechanism: open.
- **Receipt bag size at scale** (v0.4). A mature peer accumulates thousands
  of receipts over months. Recomputing `A` from the full bag at every
  bond-admission is `O(receipts)`. Optimisation: the peer presents a
  compressed time-bucketed summary plus selective opens of the N most-
  recent receipts for verifier spot-checks. Mechanism not fully specified
  beyond "selective disclosure per §Selective disclosure of receipts"
  (v0.5 specifies the Merkle-bag-root mechanism and the compact-summary
  optimisation; bucket granularity calibration remains open).
- **Bonds across forks** (v0.4 ∩ RFC-0001 v0.3 ledger portability). When a
  peer's reputation is ported across a fork (RFC-0001 v0.3's load-bearing
  portability), are locked bonds also carried? Probably yes for unresolved
  bonds, with the dispute window restarting from the fork point. Not
  explicitly specified.

**v0.5 — closed by the Round 1 amendment** (cross out from this list,
recorded here for traceability):

- ~~Recovery from network partition~~ → **closed in v0.5** §Partition
  recovery (equivocation slashing + Σ-T fork choice + social-Schelling
  fallback for balanced cases).
- ~~State growth / chain pruning~~ → **closed in v0.5** §G-Set pruning
  and late-joiner protocol (post-epoch-confirmation pruning, L2 Merkle
  roots as the canonical record, archive nodes for on-demand
  retrieval).
- ~~Bond admission atomicity under L1 propagation latency~~ → **closed
  in v0.5** §Atomicity (race-safe protocol with δ_admission delay +
  deterministic BLAKE3 tie-break; no NTP dependency).
- ~~Receipt bag size at scale~~ → **partially closed in v0.5**
  §Selective disclosure of receipts (Merkle-bag-root + compact-summary
  mechanism; bucket granularity remains open).

**v0.5 — new open items**:

- **Archive-node NCS reward formula.** Long-term retention of pruned
  proofs is incentivised via NCS reward per served retrieval. Exact
  formula (per-byte? per-query? scaled by proof age?) is calibratable
  and unmeasured.
- **`δ_admission` calibration per act class.** Default 5 seconds for
  time-sensitive acts (Tier 3 verification); longer for slower acts
  (PoUH committee, fork-recovery vote). Each act class needs its own
  empirical calibration on b2.
- **Compact-summary bucket granularity.** Time-bucket size for the
  receipt summary mechanism is a privacy/efficiency trade-off (smaller
  buckets → more privacy leak per audit; larger buckets → less
  granular `A` verification). Calibratable on testnet.
- **ZK selective disclosure of receipts.** Future work: replace the
  Merkle-bag-root mechanism with zero-knowledge proofs of the same
  threshold statements, eliminating the privacy-over-time leak.
  Implementable in principle (the statements are arithmetic on bounded
  scalars, not LLM inference — orders of magnitude cheaper than ZKML).
  Not specified in v0.5; the door is left open for a future amendment
  when ZK threshold proofs are mature enough to compete with the
  simpler selective-disclosure mechanism on cost.
- **Censorship resistance during partition.** During a partition,
  equivocation-resistant validators on the eventual losing fork may
  unintentionally censor honest activity on the winning fork (they
  cannot sign blocks they cannot see). The protocol does not currently
  distinguish "censorship by malice" from "censorship by partition."
  Probably treated identically — open whether that is acceptable or
  whether partition-induced unavailability should be excused.

---

## Adversarial considerations

The familiar list, with the v0.2 graceful-degradation argument made explicit
where it changes things.

**The systematic L1 forger.** An adversary mints fake fault proofs hoping
honest peers slash innocent victims. Defeated by `VERIFY`: every peer running
the function returns "invalid" on a forgery. No widely-distributed forgery
attack is possible.

**The eclipse attacker.** Surrounds a victim peer in the gossip overlay so
real fault proofs never reach them. Mitigations: bounded by the anti-eclipse
peer-selection rules in RFC-0005; the victim is *one* peer, and the proof
still propagates through the rest of the network; the binding constraint
`T_prop < T_beacon` ensures even an eclipsed victim is removed from the
verifier rotation faster than its slash can be exploited.

**The L2 committee bias attack.** An adversary tries to influence VRF
seeding to land its peers in many committees. Defence: the seed combines
previous block hash, height, and an external entropy beacon (drand or
equivalent). Manipulating the seed requires manipulating *external* entropy,
which is much harder than corrupting the chain.

**The cabal of L2 validators.** A coordinated subset censors specific
pattern findings. **This v0.2 architecture limits the blast radius:** they
cannot fabricate slashes; they can only delay confirmation, and the next
epoch picks up the backlog. A sustained cabal large enough to suppress
patterns indefinitely is also large enough to control the chain, but it
*still* cannot reverse Layer 1 slashes. This is the graceful-degradation
property we designed for.

**Spam attacks at L1.** Address above: VERIFY-then-relay + accountability +
per-attested-identity rate limits.

**Spam attacks at L2.** Submitting bogus L2 transactions requires paying a
small NCS fee. The fee is small enough not to discourage legitimate
reports, large enough to make spam economically prohibitive at scale.

**The hardware vendor compromise.** Discussed in RFC-0005. Mitigation:
multi-vendor diversity in committees; weighting that downweights any
single-vendor majority. Even a total compromise of one vendor's
attestation-signing key cannot fabricate L1 fault proofs (those are
self-authenticating against the *subject's* key, not the vendor's), but it
*can* mint fake identities that flood committee selection. Multi-vendor
weighting is the defence.

---

## A reference sketch

In the habit of the other RFCs — small enough to argue with, not
production-grade:

```python
# ────────── Layer 1: the consensus-free CRDT ──────────

def on_receive_proof(self, π):
    if not verify_fault(π, π.subject_pubkey):
        self.rate_limit(self.sender_of(π))
        return
    if π in self.proof_set:
        return                          # idempotent
    self.proof_set.add(π)
    self.apply_slash_locally(π)        # immediate, no consensus needed
    self.gossip(π)                     # forward to peers

def is_slashed_locally(self, peer_id):
    return any(p.subject == peer_id for p in self.proof_set)

def tenure(self, peer_id, receipts, now):
    if self.is_slashed_locally(peer_id):
        return 0.0
    by_bin = group_by_time_bin(r for r in receipts
                               if countersigned(r) and fresh_nonce(r))
    return sum(min(KAPPA * BIN, total) * exp(-DELTA_T * (now - t))
               for t, total in by_bin)

# ────────── Layer 2: the PoUH chain as ordered witness ──────────

def epoch_summary(self, committee, cutoff_time):
    # Committee observes the gossiped CRDT at cutoff.
    visible_proofs = [π for π in self.proof_set
                      if π.signed_at <= cutoff_time]
    confirmed_root = MerkleRoot(visible_proofs)

    findings = []
    for peer_id in unique_subjects(visible_proofs):
        for rule in PATTERN_RULES:                   # protocol-defined
            count = count_events(peer_id, rule.window,
                                 rule.event_filter, visible_proofs)
            if count >= rule.threshold:
                findings.append(PatternFinding(peer_id, rule, count))

    return EpochSummary(
        epoch=self.epoch,
        cutoff_time=cutoff_time,
        confirmed_proofs=confirmed_root,
        pattern_findings=findings,
    )

# Note: the committee NEVER fabricates a fault proof. It can only attest to
# what was already in the visible CRDT. A reader verifying an epoch summary
# can independently re-verify every proof referenced by confirmed_root —
# nothing the committee says is taken on faith beyond the existence of those
# proofs in the gossip overlay at cutoff_time.
```

The two-layer shape is the whole architecture: **trust is destroyed
immediately and by anyone holding a proof; ordered patterns are confirmed
slowly and by a rotating committee; the committee can never fabricate.**

---

## How to contribute to this document

Open an issue on `Ants-Community/ants` with the tag `rfc-0004`. The three
familiar paths: clarity PRs (merged quickly); design-question issues
(discussed in the issue, then resolved with a PR or escalated to Matrix);
amendments drafted as patches with rationale. Decisions land in
`spec/CHANGELOG.md`.

The two claims we will most defend, and most want broken: that **the
two-layer architecture preserves the trust-closed property** of the v0.1
corpus (the recursion bottoms out at the bounded-time social-coordination
assumption, not at the witness committee); and that **the chain's
compromise model is degradation, not corruption** — a malicious committee
cannot fabricate slashes, only delay them. If either is wrong, this is the
document where it is most worth being wrong out loud.

The document is CC0. Quote it. Disagree with it in public. Fork it.

---

*Drafted by the ANTS founding contributors, May 2026.
A reputation system in two layers — the one that says no, and the one that
remembers who said it.*
