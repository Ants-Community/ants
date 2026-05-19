# RFC-0003 — Reputation

**Status:** Draft · v0.1
**Topic:** The reputation layer of ANTS — what the network remembers, where, and why it needs no consensus to do it.
**Audience:** You, if you are willing to disagree in writing.

---

## What does the network remember?

[RFC-0001](./RFC-0001-barter.md) gave each peer a private ledger of who owed
whom. [RFC-0002](./RFC-0002-verification.md) needed a verifier set selected from
"the reputation set." [RFC-0004](./RFC-0004-identity.md) needed earned weight, a
genesis schedule, attributable faults, and a fork-time ranking — all of them
*global*.

Those two needs are in direct contradiction, and this document is where the
contradiction is paid for. RFC-0003 is the **keystone**: it is the smallest layer
by mechanism and the largest by load, because it is where the deliberate locality
of the economy meets the demanded globality of verification and identity — and it
must reconcile them **without importing the consensus, the blockchain, or the
authority the manifesto refuses.** It is also the document where we can finally
state, and prove, where the whole architecture bottoms out.

---

## The central tension

RFC-0001 was emphatic, and right, about its ledger:

> *"Not a blockchain. Not a shared store. A plain, local, append-only log… There
> is no aggregate, no leaderboard, no top-10… each peer's view is private to it,
> and that's deliberate. There is no block confirmation."*

That locality is load-bearing for RFC-0001: no central authority, no global score
to game, privacy of the ledger. We will not weaken it.

But RFC-0002 and RFC-0004 ask for the opposite:

- RFC-0002's R3 selects auditors from `ReputationSet` by a public deterministic
  function — that presumes a *globally agreed* set, not ten thousand private
  bilateral views.
- RFC-0004's amortised identity, its honest-majority assumption, its genesis
  schedule, and its fork-time successor ranking are all *global, public,
  consistent* objects.

So: **RFC-0001 needs reputation local and private; RFC-0002 and RFC-0004 need it
global and consistent; the manifesto forbids the consensus that would normally
bridge them.** This is the impossibility this layer must dissolve.

---

## The resolution: two ledgers, on different axes

The error is to think there is one ledger. There are two, and RFC-0003's first job
is to declare that **they are not the same object**:

1. **The economic ledger stays exactly as RFC-0001 specified** — local, private,
   per-pair, no consensus. It governs choke/unchoke and serving priority. RFC-0003
   does not globalise it; doing so would break RFC-0001.
2. **A separate, minimal, global register records only what carries its own
   proof.** Its governing principle, stated once and adhered to without exception:

> The global layer records only **self-authenticating negative facts.** Everything
> positive is either local, default-granted, or computed once, deterministically,
> at fork time.

Everything below is the consequence of taking that principle seriously.

---

## The global layer needs no consensus

Every fault proof `π` is self-authenticating: `VERIFY(π) → {valid, invalid}` is a
**deterministic, context-free, pure function**, evaluable offline by anyone holding
the subject's public key (committed at genesis, RFC-0004). The consequence is
decisive: **no two honest peers who run VERIFY can disagree about any individual
proof.** If they held the same set of proofs, they would compute the same slash
status for everyone.

So the global problem is **not consensus** (agree on a value *despite
disagreement*). It is **set convergence** of monotone, idempotent, commutative
facts — a grow-only-set CRDT of self-verifying elements. A Byzantine node cannot
inject a false slash (VERIFY rejects it) nor un-slash one (append-only, the
manifesto's word). Its only powers are:

```
W1  withhold / eclipse — keep a proof from reaching an honest victim
W2  propagation delay  — the proof spreads, but not yet everywhere
W3  invalid-proof spam — flood VERIFY with garbage to exhaust it
```

The only security parameter is therefore the **propagation window**. Epidemic
gossip over `N` honest peers, fanout `f`, round interval `Δ`:

```
   T_prop  ≈  c · Δ · log N      (with high probability)
```

After `T_prop`, a slashed identity is globally dead, permanently. The window is
logarithmic in network size — but the bound **holds only under an anti-eclipse
assumption** (the honest subgraph is not partitionable around the detector or the
victim). Without it the guarantee is vacuous, exactly as RFC-0002's Type-I bound is
vacuous without its null and RFC-0004's security is vacuous without honest-majority.
We state the assumption; we do not hide it.

### The structural win

One asymmetry makes the consensus-free design actually work:

> A monotone, self-authenticating fact needs **one honest path, ever — not an
> honest majority.**

Consensus needs an honest majority. Epidemic dissemination of a self-verifying
monotone fact succeeds if there exists *a single* honest path from detector to
victim within the window. To suppress it the adversary must achieve a **full vertex
cut** of the honest subgraph around the victim — not merely out-number it. This is
an exponentially weaker assumption than honest majority, and quantifiably cheaper to
guarantee. It is what lets a global state exist without consensus at all.

### The constraint that actually binds

The window that matters is not "until `N` peers know." It is "until the peers the
RFC-0002 beacon is about to pair with the slashed identity know." That yields the
binding cross-layer constraint:

```
   T_prop  <  T_beacon
   (a slash must outrun RFC-0002's verifier-rotation cadence;
    otherwise a freshly-slashed colluder is re-selected before
    the network knows)
```

A quantified trade falls out, not a hand-wave: faster gossip (smaller `Δ`, larger
`f` → bandwidth cost) versus slower beacon rotation (worse fraud-detection latency,
which couples back to RFC-0002's challenge rate `p`). DoS via invalid proofs (W3)
is **rate-limited and made accountable** — relaying garbage is itself an
attributable fault — but not *prevented*: there is an irreducible `O(f)`
verification cost per novel candidate. Resistance is accountability, not immunity.

---

## Tenure: granted by default, destroyed by proof

RFC-0004 needs a slow, Sybil-resistant, rate-capped reputation component `T`
(tenure) gating verifier eligibility, alongside the fast `A` (active) component
governing weight. But tenure is a *positive* quantity, and the global layer carries
only *negative* self-authenticating facts. Positive history is exactly what would
need consensus. The resolution:

- **Positively**, tenure is never stored or agreed globally. It is *recomputable*
  by anyone, on demand, as a deterministic function of the holder's bag of
  **counterparty-countersigned receipts**:

  ```
  T_X(τ)  =  Σ_receipts  e^{−δ_T (τ − t)} · clip_κ( per-identity accrual / time )
  ```

  Everyone evaluating X's receipts computes the same `T_X` (deterministic, like
  VERIFY). The cap `κ` lives *inside* the function — receipts are time-binned,
  per-key accrual clipped at `κ` — so presenting ten thousand receipts for one
  minute yields no more than `κ·(one minute)`. This is the precise mechanism of
  *time as the resource that cannot be bought*: you cannot rent more clock.
- **Negatively**, tenure is zeroed by a self-authenticating fault proof propagated
  through the layer above.

The hard attack is **collusive receipt-farming**: an adversary's identities
countersign each other's receipts for work never done. Nothing about such a receipt
is attributably false *at accrual* — this is the recursion, surfacing again. The
resolution is not to prevent it (impossible without consensus) but to make farmed
tenure **strategically inert**, via a forced trilemma:

- *never used* → zero influence. Weight in any round is governed by `A`, not `T`;
  a farmed-tenure identity with no recent real work has no `A`.
- *used honestly* → it **is** the useful work the network wanted. The
  amortised-identity inversion of RFC-0004, one level down.
- *used to collude* → it must produce RFC-0002 commitments; collusion produces an
  attributable fault (transitive verifiability), which propagates and zeroes the
  tenure and the identity.

There is no branch with positive attack value — **except** a bounded residual we
state honestly: detection is probabilistic (RFC-0002's `p`), so a colluder gets
roughly `1/p` actions before the fault lands. That value is governed by RFC-0002's
deterrence inequality and RFC-0003's propagation window. Which is the point:

> Tenure introduces **no new security parameter.** Its integrity is exactly
>
> ```
> tenure integrity  ⟺  (RFC-0002 deterrence inequality)  ∧  (T_prop < T_beacon)
> ```
>
> — the conjunction of two constraints already derived elsewhere.

This is the keystone behaving like a keystone: it does not add a wall, it
distributes the load onto members that already exist. The one new requirement is
disciplinary, not structural: receipts must be counterparty-countersigned, bound to
a fresh nonce from the issuer, and attributably timestamped (a back-dated receipt is
equivocation against the issuer's signed clock). It is the same anti-equivocation
discipline RFC-0002 and RFC-0004 already use, reused.

---

## The fork-time ranking, and where it ends

RFC-0004's fork-recovery needs the successor trustee set to be the top earned-
reputation peers at the pre-fault cut. A *global positive ranking* is the one thing
the consensus-free model cannot natively provide: it is relational, it needs a
complete view, and positive evidence is pulled on demand, not pushed. Naively, it
is impossible — and we say so plainly.

It is rescued by changing what is ranked:

1. **Self-nomination.** At fork time, any peer that wishes to be a successor
   *publishes* a self-authenticating tenure attestation (its receipt bag,
   timestamped ≤ the cut). Positive evidence becomes *push, not pull* — the
   candidate is incentivised to propagate it — and self-authenticating, so it
   converges through the same gossip as faults.
2. **A hard cutoff** at `t_fault − T_prop`: a nomination counts only if it is one
   full propagation window old at the cut. Under the anti-eclipse assumption every
   honest peer then provably holds every valid nomination. The ranking problem is
   thereby **reduced to the already-solved convergence problem above**, at the cost
   of a timing margin — no new assumption.
3. **A total deterministic order** with a cryptographic tie-break keyed to
   `t_fault`, so the same nominee set yields exactly one ordering — no wiggle room
   to fragment the honest set.

And the residual, stated exactly: the cutoff guarantee holds *only under
anti-eclipse*. If the adversary can partition the honest subgraph during the
nomination window, honest peers genuinely disagree about the nominee set, the
ranking diverges, and **there is no consensus-free way to resolve it** — resolving
disagreement about a positive set *is* consensus. At that point recovery falls to
**exactly the social-Schelling fallback RFC-0004 already named as the ultimate
floor**: the human community, using the manifesto, the public fault proof, and the
largely-converged nominee set as the focal point. No deeper. No shallower. The loop
closes.

The protocol must therefore *declare its own non-algorithmic fallback in writing*:
self-nomination, the `t_fault − T_prop` cutoff, the deterministic order, **and** an
explicit, manifesto-anchored social-Schelling rule for the partition case. That
explicitness is the entire difference between ANTS and the systems that have a
social trust root and pretend they do not.

---

## Where ANTS bottoms out

This is the keystone, so this is where it is said. Trace every hard residual in the
four RFCs to its end:

```
RFC-0002  Type-I rigour          ⊥  honest-majority verifier set        → RFC-0004
RFC-0004  amortised identity     ⊥  genesis bootstrap → credible fork → bounded social coordination
RFC-0003  global layer           ⊥  anti-eclipse → Sybil-resistant peers → RFC-0004  (recursion)
RFC-0003  tenure                 ⊥  (RFC-0002 deterrence) ∧ (T_prop < T_beacon)   [composes existing]
RFC-0003  fork-time ranking      ⊥  anti-eclipse + cutoff; under partition → the social floor
```

Every residual terminates in **exactly one of two places**: a *quantified
cross-layer parameter constraint* (the deterrence inequality, `T_prop < T_beacon`,
`e ≥ T`, the `(A,T,δ_A,δ_T,κ)` spine), or *the single bounded-time
social-coordination / credible-fork-threat assumption.* There is no third kind of
foundation. Critically, **nothing bottoms out shallower** than that one social
assumption — no hidden trusted party, no vendor, no token, no permanent authority.

So the recursion **closes**. It terminates; it does not diverge into an infinite
regress of trust, and it does not secretly rest on a central point. ANTS is a
recursive object **with a fixed point**, and the fixed point is named, minimal,
bounded in time, and fork-recoverable.

This is not "trustless." We will not claim that; it would be the project lying to
itself. It is **trust-closed**: the recursion resolves to a single explicit
assumption, and that assumption is the smallest, most visible, most recoverable one
we know how to build inside the manifesto's refusals. Whether trust-closed is enough
is the question the corpus puts to its reader. It does not assume the answer. It
refuses, on principle, to pretend the question is not there.

---

## What we have not figured out yet

- **The anti-eclipse assumption is the new load-bearer**, and it recurses:
  anti-eclipse needs Sybil-resistant peer selection, which is RFC-0004, which needs
  this layer. The one-path-not-majority asymmetry makes the required dose small —
  but "small" is not "zero," and we have no field data.
- **`T_prop < T_beacon` is asserted, not measured.** Real gossip latency under a
  partially adversarial topology is the number a testnet must produce, alongside
  RFC-0002's `e`.
- **The partition case has no algorithmic answer**, by construction. The social
  fallback is honest but fragile to apathy, fragmentation, and social-layer
  capture — the same erosion RFC-0004 names.
- **Censorship remains non-attributable** (inherited from RFC-0004): a genesis or
  high-tenure actor that withholds rather than equivocates produces no proof.
- **Ledger portability is now load-bearing.** RFC-0001 deferred it to "v0.3";
  RFC-0004's fork-recovery makes it a security parameter of the social root. RFC-0001
  must be amended, not patched here.

---

## What comes next

With this draft the v0.1 corpus exists end to end: economy (RFC-0001), verification
(RFC-0002), reputation (RFC-0003), identity (RFC-0004). They are not four documents.
They are one recursive object with one fixed point, and this is the document that
proves it.

The immediate follow-on is an **amendment to RFC-0001**: ledger portability
reclassified from convenience to security parameter, and the still-open `S_NCS`
slash-value question that RFC-0002's threshold depends on. The corpus is internally
consistent; it is not yet field-tested, and it says so everywhere it matters.

---

## A reference sketch

In the habit of the other RFCs — small enough to argue with, not production-grade:

```python
def slashed(identity, proof_set):
    # consensus-free: a pure function of self-authenticating monotone facts
    return any(verify_fault(p) and p.subject == identity for p in proof_set)

def tenure(identity, receipts, now):
    by_bin = group_by_time_bin(r for r in receipts
                               if countersigned(r) and fresh_nonce(r))
    return sum(min(KAPPA * BIN, bin_total) * exp(-DELTA_T * (now - t))
               for t, bin_total in by_bin)            # kappa-clipped, time-binned

def eligible_verifier(identity, proof_set, receipts, now):
    return (not slashed(identity, proof_set)
            and tenure(identity, receipts, now) >= TENURE_FLOOR)

def fork_successors(t_fault, proof_set, k):
    noms = [n for n in gossiped_nominations()
            if n.timestamp <= t_fault - T_PROP            # hard cutoff
            and not slashed(n.identity, proof_set)]
    noms.sort(key=lambda n: (tenure(n.identity, n.receipts, t_fault),
                             H(n.identity, t_fault)),     # deterministic tie-break
              reverse=True)
    return noms[:k] if converged(noms) else SOCIAL_SCHELLING_FALLBACK
```

The last clause is not an omission. It is the protocol stating, in its own source,
the one place it hands the problem back to the colony — because the colony, not the
code, is the floor.

---

## How to contribute to this document

Open an issue on `Ants-Community/ants` with the tag `rfc-0003`. Same three paths as
the other RFCs: clarity PRs merged quickly; design-question issues discussed then
resolved or escalated to Matrix; amendments drafted as patches with rationale.

The two claims we will most defend, and most want broken: that the global layer
genuinely needs no consensus — only set-convergence of monotone self-authenticating
facts, one honest path and not a majority — and that the architecture is
*trust-closed*: every residual terminates at a quantified constraint or the single
bounded social assumption, and nothing bottoms out shallower. If either is wrong,
this is the document where it is most worth being wrong out loud.

The document is CC0. Quote it. Disagree with it in public. Fork it — and note that
this document is, in part, the specification of how to do exactly that.

---

*Drafted by the ANTS founding contributors, May 2026.
The colony has no center. This document does not either.*
