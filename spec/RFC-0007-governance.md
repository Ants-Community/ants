# RFC-0007 — Post-v1.0 Governance: TSC Mechanics

**Status:** Draft · v0.1
**Topic:** How the Technical Steering Committee that replaces the BDFL at v1.0 actually works — composition, elections, terms, decision thresholds, conflicts of interest, transition flow.
**Audience:** You, if you might one day vote in this election or hold one of its seats.
**Depends on:** [GOVERNANCE.md](../GOVERNANCE.md), [MANIFESTO.md](../MANIFESTO.md)

---

## Why this RFC is being drafted during v0.x

The external multi-persona review of 2026-05-20 named this gap precisely:

> *"RFC-0007 (Post-v1.0 Governance) is planned, not written. GOVERNANCE.md
> says 'TSC with 7 seats, 2/3 majority on canonical artefacts, unanimous on
> governance changes' but the operational details (who elects the 3
> subsystem maintainer seats? What qualifies as a 'subsystem'? How long is
> a term?) do not exist. It needs to be written before sunset, not after,
> and written in v0.x with time for iteration."*

The criticism is correct. GOVERNANCE.md specifies the *intent* of the TSC
(7 seats, decision thresholds, term length) but not the *mechanics*. A
mechanism people will rely on at v1.0 must be specified before they need
to rely on it. Specifying it during v0.x — when contributors can argue
without anything being at stake — is the only fair way to do it.

This RFC promotes itself from `Planned` to `Draft v0.1`. It will iterate
during v0.x as contributors find problems with it. By v1.0, the TSC
mechanics in this document are what the first election runs against.

---

## TSC composition: seven seats

The Technical Steering Committee has **seven seats**. The composition
balances technical depth (the subsystem maintainers) with broad
representation (the at-large contributors) plus continuity (the chair).

| Seat type | Count | Elected by | Term |
|---|---|---|---|
| Subsystem maintainer | 3 | Contributors active in that subsystem | 2 years, staggered |
| At-large contributor | 3 | All contributors with merged work | 2 years, staggered |
| Chair | 1 | TSC members from among themselves | 1 year, renewable once consecutively |

A maximum of 4 seats turn over in any one renewal cycle, so the TSC
preserves majority continuity across elections.

### The three subsystems

The protocol decomposes into three subsystem domains for the purpose of
this election. The boundaries are intentionally chosen so that each domain
has comparable scope, comparable contributor population, and a
self-contained set of RFCs.

| Subsystem | RFCs | Concerns |
|---|---|---|
| **Verification & Numerics** | RFC-0003, RFC-0009, RFC-0011 | inference correctness, e-process, canonical kernels, formal model |
| **Reputation & Identity** | RFC-0004, RFC-0005, RFC-0008 (crypto primitives) | CRDT, PoUH chain, A-as-bond, TEE attestation, wire primitives |
| **Cache, Economy & Coordination** | RFC-0001, RFC-0002, RFC-0006, RFC-0010 | barter, semantic cache, payment terms, bootstrap |

The BDFL may revise the subsystem boundaries during v0.x via amendment to
this RFC. After v1.0, boundary revisions require a 2/3 TSC majority.

### Why three subsystems and not four or six

Three is small enough that each subsystem has enough contributors to make
a meaningful election, and large enough that the technical concerns don't
all overlap. Six would split contributor populations too thin in the
early-network phase; one or two would over-concentrate authority. The
boundaries are revisable; the count is also revisable; the goal is "the
smallest decomposition that gives each major technical concern a
representative."

---

## Election mechanics

### Subsystem maintainer seats (three seats, one per subsystem)

**Eligibility to vote**: every contributor who has had work merged in
that subsystem's RFC files or reference-client components in the trailing
**12 months** at the time of nomination opening.

**Eligibility to stand**: every voter for that subsystem is eligible to
be a candidate.

**Nomination period**: 14 days. Anyone may nominate any eligible
candidate (including themselves). Nominations include a 200-word
candidate statement.

**Election**: 14 days of voting. Each voter has one vote per subsystem
seat (so subsystem-A voters vote only for the subsystem-A seat). Single
transferable vote (STV) — no, simpler: **single non-transferable plurality
vote with run-off** if no candidate exceeds 50% in the first round. The
top two then face off in a 7-day run-off.

**Term start**: immediately after the run-off, or the day after the first
round if no run-off needed.

### At-large contributor seats (three seats)

**Eligibility to vote**: every contributor who has had any work merged
(any RFC, any reference-client component, any brand/governance/process
artefact) in the trailing **12 months**.

**Eligibility to stand**: every voter is eligible.

**Nomination period**: 14 days. Anyone may nominate.

**Election**: 14 days of voting, **single transferable vote (STV)**.
Three winners. STV ensures that minority preferences within the
contributor body get represented; this matters here because the at-large
seats are intentionally the "broad community" voice.

**Term start**: same as subsystem.

### The chair seat

**Transitional period (first 24 months post-v1.0)**: the outgoing BDFL
serves as transitional chair. This is a hard 24-month limit, not
renewable.

**After the transitional period**: the chair is elected by the TSC
members from among themselves for **1-year terms, renewable once
consecutively** (maximum 2 consecutive years; longer service requires a
1-year break before another term).

**Chair role**: presides over TSC meetings, sets agendas, breaks ties on
simple-majority votes only. The chair has **no veto** and no enhanced
voting power on supermajority matters.

**Removal**: the TSC may remove the chair by 2/3 vote at any time. A
removed chair retains their underlying seat (subsystem maintainer or
at-large) for the rest of its term.

---

## Decision-making thresholds

The TSC's decisions fall into three classes, each with a different
threshold per GOVERNANCE.md and refined here.

| Class | Examples | Threshold | Notes |
|---|---|---|---|
| **Routine** | meeting cadence, working-group formation, contributor-status acknowledgement, minor process | simple majority (4 of 7) | Chair breaks ties. |
| **Canonical artefact** | RFC amendments, reference client merges, brand asset changes, embedding model upgrades (per RFC-0002 v0.2 §Governance), domain custody changes | 2/3 majority (5 of 7) | No tiebreak — supermajority required, chair has no enhanced vote. |
| **Governance** | amendments to this RFC, amendments to GOVERNANCE.md, dissolution or restructure of the TSC | unanimous (7 of 7) | Highest bar; intended to be rare. |

A vote that fails to reach its threshold remains open: the proposal is
**not** rejected; it is **not adopted**. The proposer may revise and
resubmit, or withdraw.

### Quorum

Routine and canonical-artefact decisions require at least **5 of 7**
seats voting (yes, no, or abstain — abstention counts toward quorum but
not toward the threshold). Governance decisions require all 7.

A seat that is vacant (resignation, removal, death) counts as not
present for quorum until filled. Off-cycle elections to fill vacant seats
are triggered by the chair within 30 days; expedited timelines (7-day
nomination, 7-day election) apply.

---

## Conflicts of interest

A TSC member with a financial or commercial interest in a specific
decision must **disclose** the conflict at the start of any deliberation
and **recuse** themselves from the vote. Recusal means:

- The member's seat counts as not present for quorum on that decision.
- The thresholds are computed against the remaining six (or however many
  uninvolved) seats. For canonical-artefact decisions this means 4 of 6
  is the supermajority (still 2/3 rounded up); for governance, all
  remaining seats must vote unanimous (i.e., the vote can proceed only if
  the non-recused members all agree).

Examples of conflict warranting recusal:

- Employee or owner of a commercial operator (e.g., Anthropic, OpenAI)
  with a paid service running on the network — on decisions about
  RFC-0006 commercial Payment Terms or cross-economy settlement.
- Holder of significant cloud-TEE rental contracts — on decisions about
  Sybil-resistance economics (RFC-0005).
- Maintainer of the canonical embedding model upstream (e.g., a BAAI
  employee) — on decisions about embedding model upgrade governance
  (RFC-0002 v0.2).

The contributor body may petition for a member's recusal on a specific
decision via signed-issue petition by ≥10% of contributors eligible to
vote in the relevant subsystem (for subsystem seats) or eligible to vote
at-large (for at-large seats). The TSC reviews the petition; the
petitioned member may dispute or accept. A disputed recusal is decided
by the TSC by simple majority (with the petitioned member recused from
that meta-vote).

### Off-cycle removal

A TSC member who is widely considered to be acting in bad faith may be
removed by:

- 2/3 of the **remaining** TSC seats (the petitioned member recused), AND
- A signed petition by ≥25% of contributors eligible for the relevant
  seat class.

This is intentionally a high bar. The intent is recovery from genuine
abuse, not retaliation for unpopular decisions.

---

## Transition flow at v1.0

The single irreversible event from BDFL to TSC governance happens in a
strictly-ordered sequence:

1. **BDFL declares v1.0** by signing a CHANGELOG entry attesting that all
   five GOVERNANCE.md conditions are met. This declaration starts the
   transition clock.

2. **60-day nomination period** (longer than the 14-day routine cycle —
   the v1.0 transition is unusual and warrants extra deliberation).
   - Subsystem maintainers are nominated by their subsystem communities.
   - Contributors-at-large self-nominate or are nominated by anyone.

3. **30-day election period**. Subsystem maintainer seats by plurality
   with run-off; at-large seats by STV. Conducted via the
   reputation-attested identity system (RFC-0005); each contributor's
   vote is signed by their identity and publicly verifiable.

4. **First TSC convenes**. The outgoing BDFL serves as chair for 24
   months. The new subsystem-maintainer and at-large members take office.

5. **The BDFL's pre-v1.0 special authorities expire**. The CHANGELOG
   records "BDFL period ended" with the date.

6. **24 months later**: the outgoing-BDFL chair role expires. The TSC
   elects a new chair from among its members per the normal mechanics.
   The former BDFL retains contributor status, may run for TSC seats by
   election like any other contributor, holds no residual special
   authority.

The transition is **irreversible** by design. A future TSC may not
re-instate a BDFL period; doing so would require amending GOVERNANCE.md,
which requires unanimous TSC vote on a governance matter and is therefore
extremely difficult.

---

## Amendment process for this RFC

| Period | Process |
|---|---|
| **During v0.x** (before v1.0 transition) | BDFL amends after a **4-week public comment period** (extended from the standard 1–3 weeks because governance amendments are consequential). The 4-week minimum is non-negotiable downward. |
| **After v1.0** | Per GOVERNANCE.md: **unanimous TSC vote** following a **60-day public comment period**. Highest bar in the protocol. |

The "high cost to amend" property is intentional. A governance document
that can be changed easily is not a governance document.

---

## What we have not figured out yet

The places this RFC most needs to be argued with:

- **Exact subsystem boundaries.** RFC-0009 (canonical numerics) is
  currently in Verification & Numerics, but it sits at the seam with
  Reputation (because canonical-recipe output drives bond admission via
  Y_canon). If RFC-0009 ends up effectively shared between subsystems,
  the boundary needs revisiting.
- **Voter eligibility for contributor seats.** "Merged work in trailing
  12 months" is the headline rule, but: does a translation count? Does a
  single typo fix? Does an essay or talk count? Probably "any merged PR,
  any tagged issue resolution, any RFC amendment" — but the exact rule
  is open and a contributor body should weigh in.
- **Anti-Sybil for elections.** Voters must be attested identities per
  RFC-0005, but each identity is one CPU — does that mean a researcher
  with three machines gets three votes? Probably yes, since the cost of
  the three machines is the Sybil-resistance economic floor. This may
  feel unfair to "humans behind one identity"; the alternative (some
  off-chain personhood verification) violates the manifesto. The
  trade-off is honest but worth explicit acknowledgement.
- **Foundation treasury governance.** The Foundation that the manifesto
  Thesis 16 promises does not yet exist legally; when it does, the
  relationship between its budget, its custodians, and the TSC must be
  specified. This may warrant a follow-up RFC-0012 once the legal
  foundation exists (per the GOVERNANCE.md "foundation appears post-
  proof-of-feasibility" stance).
- **Off-cycle election triggers.** Resignation, removal, incapacitation
  — all covered above for the existing seats. Catastrophic failures (3
  seats vacant simultaneously) need a continuity-of-governance protocol
  not yet specified.
- **TSC meeting cadence and venue.** Open question: how often does the
  TSC meet (monthly? quarterly?), how public are deliberations (every
  decision logged in a public minutes file, definitely; live-streamed
  deliberation, probably no), what are the working-group protocols
  delegated under the TSC.
- **Liaison roles to outside bodies.** The W3C/IETF/IEEE world has
  protocol-stewardship patterns the ANTS TSC will eventually interact
  with. Whether to designate a specific liaison seat or rotate per matter
  is not yet specified.

---

## How to contribute

Open an issue on `Ants-Community/ants` with the tag `rfc-0007`. Most
valuable contributions during v0.x:

- **Stress-testing the mechanics.** "What happens if X resigns the day
  after their seat fills? Can the run-off be gamed?" These are the
  scenarios this RFC needs to be robust to.
- **Critiques from existing OSS-foundation practitioners.** The patterns
  of Linux Foundation, Apache Foundation, Rust Foundation, OpenStack
  Foundation are well-documented; their successes and failures inform
  what RFC-0007 should not repeat.
- **Subsystem-boundary proposals.** If the three-subsystem decomposition
  has fault lines, name them.

The document is CC0. Adopt the mechanics for your own foundation if
useful.

---

*Drafted by the ANTS founding contributors, May 2026.
The colony that endures is the one that decides how to decide before it has to.*
