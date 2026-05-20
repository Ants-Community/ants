# RFC-0007 — Post-v1.0 Governance: TSC Mechanics

**Status:** Draft · v0.2
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

**Agenda-setting authority is real authority.** *Added Round 4 EDGE
to address the concern raised by the multi-persona Round 2 review
(OSS-governance persona).* The chair's agenda-setting power — what
gets discussed, in what order, with how much time — is a meaningful
form of influence that does not appear in any vote tally. Specifying
this honestly: during the 24-month transitional-chair period, the
former BDFL retains agenda-setting authority over TSC deliberations.
The BDFL therefore does not *disappear* at v1.0 declaration; the
BDFL disappears 24 months later, when the transitional chair role
expires. This is intentional (continuity of expertise during the
trickiest period of the project's governance arc) and is named here
rather than left implicit. The agenda-setting authority is bounded:

- **No unilateral decisions.** Setting the agenda is not voting; the
  TSC still decides. A chair who agenda-sets pathologically — burying
  inconvenient items, prioritising aligned ones — is subject to the
  same §Off-cycle removal mechanics that apply to every other TSC
  seat. The 2/3-remaining-seats threshold for chair removal (per the
  §Removal sub-section above) is the structural check.
- **Public agendas.** All TSC meeting agendas are published before
  the meeting per §TSC meeting cadence (open question). A buried item
  remains buried in public, where contributors can call it out.
- **Hard time limit.** 24 months is a hard cap, not renewable. After
  the cap, the chair role rotates per the normal mechanics, and the
  outgoing BDFL retains contributor status only.

The honest framing: agenda-setting is the place where "continuity of
the BDFL's working knowledge" most plausibly translates into
disproportionate continuing influence, even in a fully-functioning
TSC. The chair-removal threshold and the public-agenda discipline are
the protocol's two structural responses; the time limit is the third.
There is no scenario where the chair's agenda authority extends beyond
24 months under this RFC.

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
- A signed petition by **at least the size-scaled threshold** of
  contributors eligible for the relevant seat class, defined as
  `max(PETITION_FLOOR, min(PETITION_CEILING, ⌈0.25 · N_eligible⌉))`
  where `N_eligible` is the count of contributors eligible to vote for
  the seat in question.

#### Why size-scaled and not flat-25%

*Added Round 4 EDGE to address the concern raised by the multi-persona
Round 2 review (OSS-governance persona): the static "25% of eligible
contributors" threshold scales badly in both directions — too easy at
N_eligible = 50 (13 signatures suffice; a coordinated minor faction can
mount a petition), too hard at N_eligible = 10,000 (2,500 signatures is
impractical and effectively removes the petition path).*

The scaling formula has three regimes:

- **Small-network regime (`N_eligible ≤ PETITION_FLOOR / 0.25`).** The
  floor binds; a petition requires at least `PETITION_FLOOR`
  signatures regardless of how small the contributor body is. This
  prevents a 4-person faction from being able to mount a credible
  removal in a 16-person network.
- **Linear regime (`PETITION_FLOOR / 0.25 < N_eligible <
  PETITION_CEILING / 0.25`).** The 25%-of-eligible rule applies
  directly. This is the regime the static rule worked correctly in.
- **Large-network regime (`N_eligible ≥ PETITION_CEILING / 0.25`).**
  The ceiling binds; a petition requires at most `PETITION_CEILING`
  signatures. This makes the petition path practically reachable in a
  10,000-peer network without making it trivial — `PETITION_CEILING`
  is large enough that a genuine cross-cutting coalition is required.

Default values (calibratable by amendment to this RFC):

- **`PETITION_FLOOR = 12`** — enough signatures to be a real coalition,
  not 4 friends.
- **`PETITION_CEILING = 150`** — large enough to require non-trivial
  cross-cutting agreement, small enough to be realistically reachable
  in a mature contributor body.

The thresholds may be revised once the contributor body is large
enough to test the regime boundaries; the formula structure
(floor / linear / ceiling) is the contribution this section makes
and is intended to be stable across recalibrations.

This is intentionally a high bar across all three regimes. The intent
is recovery from genuine abuse, not retaliation for unpopular
decisions.

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

## Catastrophic transition scenarios

*Added Round 4 EDGE to address the strongest residual concern of the
multi-persona Round 2 review (OSS-governance persona): "RFC-0007 is
written as the governance document of a mature network. It lacks the
'transition stress test' version — the failure scenarios during the
first 24 months post-v1.0 are not pathological; they are expected in a
system with 7 people and zero operational precedent."*

The first 24 months post-v1.0 are the period of highest governance
risk: the BDFL has stepped back from decision authority but remains
chair (with the agenda-setting caveat above); the TSC members have no
operational precedent for working together; the contributor body has
not yet experienced an off-cycle election. This section specifies
continuity protocols for the four scenarios that are not pathological
but *expected*.

### S1 — Transitional chair incapacitation or sudden death

The transitional BDFL-as-chair role is filled by one person for 24
months. That person is human and may become unavailable.

**Continuity protocol:**

1. The TSC convenes within 7 days of the chair's incapacitation
   (formal medical determination, or 14 days of complete unavailability
   with no contact, whichever comes first). The longest-tenured
   subsystem maintainer presides over the convening meeting.
2. The TSC elects an **interim chair** from among its remaining 6
   members by simple majority. The interim chair serves for the
   remainder of the original 24-month transitional period — *not* the
   full 1-year non-transitional chair term.
3. The interim chair has the full chair powers (agenda-setting,
   simple-majority tiebreak), including the §Agenda-setting authority
   acknowledgment above, but is not granted the BDFL's pre-v1.0 special
   authorities (which have already expired at v1.0 declaration per
   §Transition flow step 5).
4. At the 24-month mark, the normal §The chair seat election applies:
   a fresh 1-year chair is elected.

The protocol does *not* attempt to "preserve" the original BDFL's
agenda authority across an interim chair. Continuity of governance,
not continuity of the chair, is the value being preserved.

### S2 — Multiple seats vacant simultaneously

Two or more seats becoming vacant at the same time (resignation,
incapacitation, removal) creates a quorum risk: routine and canonical
decisions require 5 of 7, so two empty seats leaves no margin.

**Continuity protocol:**

1. **Quorum holds at 5 of remaining seats** during the off-cycle
   election window. A 5-seat TSC operating with 2 vacancies functions
   normally for routine and canonical-artefact decisions until the
   vacancies are filled.
2. **Governance decisions are suspended.** Per §Decision-making
   thresholds, governance decisions require all 7 seats. A TSC with
   fewer than 7 seats may not pass governance amendments until the
   vacancies are filled. This is intentional — major structural
   decisions should not be made while the body itself is incomplete.
3. **Off-cycle elections are expedited.** Per §Quorum, the chair
   triggers off-cycle elections within 30 days. When 2+ seats vacate
   simultaneously, the elections proceed in **parallel** (not
   sequentially), with 7-day nomination + 7-day vote windows for each
   vacant seat.
4. **If 3+ seats are vacant simultaneously**, the situation is
   classified **emergency**: the chair convenes the remaining seats
   plus the most recently outgoing seat-holders (in advisory, non-voting
   capacity) within 72 hours. If the chair is among the vacant seats,
   the longest-tenured remaining maintainer convenes. Off-cycle
   elections begin immediately.

### S3 — Contested first election

The first post-v1.0 election runs on an electorate with no precedent.
A contested result — close vote, allegations of vote manipulation,
candidate withdrawal mid-vote — has no precedent to fall back on.

**Continuity protocol:**

1. **The outgoing BDFL adjudicates contested election results during
   the transitional period.** This is an additional authority granted
   to the transitional-chair role specifically to handle this scenario;
   it expires at the end of the 24-month transitional period along with
   the rest of the transitional-chair authority.
2. **The adjudication is constrained.** The BDFL may (a) declare a
   result final, (b) order a re-vote with adjusted procedures
   (extended period, observer presence, third-party verification of
   signed votes against attested identities), or (c) declare the seat
   vacant and trigger an off-cycle election under §S2 mechanics. The
   BDFL may not declare a specific candidate winner against a
   plurality of the recorded vote.
3. **The adjudication is appealable.** A contested-election adjudication
   may be appealed by ≥10% of the relevant electorate via signed
   petition. The petition triggers a 2/3-of-remaining-TSC review (with
   the BDFL-as-chair recused for the appeal). This is the only place
   in this RFC where the chair is structurally overrideable by the
   non-chair TSC.

### S4 — TSC dysfunction (deadlock, hostile minority, factionalism)

A TSC that has 4-3 splits on every canonical-artefact vote cannot
pass any 2/3 decisions. This is not pathological dysfunction; it is a
structurally possible state of a 7-person body with two genuinely
incompatible visions.

**Continuity protocol:**

1. **No artificial tiebreak.** The protocol explicitly does not grant
   any seat (including the chair) the authority to break a
   canonical-artefact deadlock by enhanced voting. A 4-3 split that
   persists across multiple meetings is a signal, not a bug.
2. **Public deliberation.** Per §Agenda-setting authority above, all
   meeting agendas and decisions are public. A deadlocked TSC's
   members publicly explain their positions, contributors weigh in
   via issue threads, and the body works through the disagreement in
   the open.
3. **Working group reference.** A deadlocked decision may be referred
   to a time-bounded working group (per GOVERNANCE.md §Structure
   beneath the BDFL) for analysis. The working group's output is
   advisory, not binding, but provides structured input to the next
   TSC vote.
4. **Right to fork remains.** Persistent dysfunction is itself a
   signal that the protocol should evolve, including the possibility
   of governance restructuring (which requires unanimous TSC, i.e.,
   the same dysfunctional body) or, in extremis, a fork that adopts
   different mechanics. The credible-fork-threat applies to governance
   the same way it applies to every other layer of the protocol.

The honest framing for all four scenarios: the protocol cannot
guarantee that the first 24 months will go well. It can specify what
the *failure modes* of those 24 months look like, so the failures are
absorbed by named continuity protocols rather than by ad-hoc
improvisation.

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
