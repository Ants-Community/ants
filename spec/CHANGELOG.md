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
