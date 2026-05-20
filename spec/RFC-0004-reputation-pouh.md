# RFC-0004 — Reputation: Two Layers, One Architecture

**Status:** Draft · v0.2
**Topic:** A reputation system in two layers — a consensus-free CRDT of self-authenticating fault proofs for individual slashing, and a small slow Proof-of-Unique-Hardware (PoUH) chain as ordered witness for pattern-based rules. The chain confirms what has happened. It does not create what happens.
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

There is no fourth branch. The marriage of (A, T, κ) plus L1 fault
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
