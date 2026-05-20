# Governance

*How decisions get made in ANTS, who makes them, and when that changes.*

---

## Preamble

A protocol that no one decides about does not stay a protocol. It becomes a museum of unfinished proposals.

ANTS adopts, for the period from v0.1 to v1.0, the same governance model that produced Linux: a single named maintainer with final authority on what enters the canonical specification and the reference implementation — accompanied by a community that contributes freely, forks at will, and steers by example.

This is not a permanent arrangement. The sunset conditions for this model are defined explicitly below. After v1.0, governance transitions to a Technical Steering Committee, with the founding BDFL serving as transitional chair.

The point of declaring this now, when ANTS has zero external contributors, is to make the model legible to anyone who arrives — so that no one wastes effort discovering, through friction, what should have been written down.

---

## The BDFL Model, v0.1 → v1.0

**Salvatore Calcerano** is the Benevolent Dictator for Life of ANTS, for the duration of v0.x. In this period he holds, exclusively and without delegation, the following authorities:

- **Specification authority.** Final merge decision on every RFC in `spec/`. Drafts and discussion are open; the final yes-or-no on what enters the canonical RFC corpus rests with the BDFL.
- **Reference implementation authority.** Final merge decision on the main branch of the reference client (when it exists).
- **Brand stewardship.** Custody of the project name, the `ANTS` wordmark, the cluster logomark, and the visual identity defined in `BRAND.md`. Forks may use the protocol freely; only the canonical project carries the name and the marks.
- **Domain custody.** Ownership of `ants.community`.
- **Repository administration.** Final say on GitHub organisation membership, repository creation, branch protection rules, and CI configuration.
- **Veto.** The right to refuse any contribution to canonical artefacts without explanation, in the rare cases where reasoned argument has been exhausted. This authority is to be used sparingly; its repeated invocation against the will of an active community of contributors is itself grounds for the BDFL to reconsider their fitness for the role.

The BDFL exercises these authorities in public, on the record, and with the explicit aim of reaching v1.0 with a governance model better than the one in which they currently operate.

---

## What the BDFL does *not* control

This list matters at least as much as the list above. It defines the irreducible openness of the protocol — the part that remains permissionless even during the BDFL period.

- **Forks.** Anyone may fork the protocol, the reference implementation, and any other artefact. Forks may not represent themselves as the canonical ANTS, but they may exist, operate, and compete. The right to fork is the right to dissent in code.
- **Network participation.** The protocol is permissionless. No party — including the BDFL — can prevent any honest peer from joining the network, serving requests, or consuming services. Hardware attestation governs Sybil resistance; nothing governs admission.
- **Implementations.** Anyone may build alternative client implementations. They are free to follow the canonical RFCs or diverge. Compliance is voluntary, judged by interoperability with peers running the reference client.
- **Discussion.** Issues, pull requests, RFC drafts, Matrix conversations, mailing lists, blog posts — all open, all permanent record, all subject to the project's Code of Conduct but otherwise unmoderated by the BDFL.
- **Independent operators.** Commercial operators serving inference on the network do so on their own terms, with their own pricing, their own SLAs, and their own liability. The BDFL has no commercial relationship with them.

---

## Structure beneath the BDFL

As the project grows, the BDFL delegates day-to-day responsibility through a Linux-style lieutenants model. These roles emerge by contribution, not by appointment; they are recognised retroactively when someone has demonstrably been doing the work.

- **Subsystem maintainers.** Trusted contributors who have proven sustained, high-quality involvement in a specific subsystem (e.g., the verification spec, the reference client networking layer, the brand assets, the documentation site). A maintainer reviews and pre-approves contributions to their subsystem; the BDFL retains final merge. Maintainers may have full commit rights to non-canonical branches.
- **Working groups.** Informal, time-bounded gatherings around a specific RFC or technical question. Anyone may join; output is captured as draft RFCs or issue threads. No formal voting.
- **Contributors.** Anyone who has had a pull request merged, an issue accepted, a translation contributed, or a published artefact (essay, talk, implementation) acknowledged. Listed in `CONTRIBUTORS.md`. No tier, no hierarchy among contributors — only a recognition that you have shown up.

There are no privileged contributors beyond maintainers. There is no "core team" with implicit decision power. The BDFL decides, maintainers shepherd, everyone else contributes.

---

## Voting weight, attested identity, and the "datacenter vs freelancer" caveat

The protocol's Sybil resistance is hardware-attested: one attested CPU is one peer identity (RFC-0005, Manifesto Thesis 10). This rule applies uniformly to participation in the network *and* to governance voting (RFC-0007 elections, contributor petitions, off-cycle removal thresholds).

A direct consequence, worth declaring rather than letting it remain implicit: **a university with a datacenter that runs 20 attested machines votes 20 times in elections that count by attested identity; a freelancer running one laptop votes once.** This is internally coherent with the manifesto's Sybil-resistance economics — the cost of hardware *is* the Sybil-resistance economic floor, and refusing it would mean either (a) reintroducing some off-protocol personhood verification, which violates Thesis 14, or (b) accepting a weaker Sybil bound. We chose hardware-attestation as the lesser ill, and the voting consequence travels with it.

This is not a defect we will eventually patch — it is a property of the design, made explicit. Mitigations the protocol *does* provide:

- **Tenure-gated influence (RFC-0004 §Tenure).** Raw identity count is not the only weight. Verifier eligibility, bond capacity, and the `Σ T_eff` fork-choice metric all use κ-rate-capped tenure, which a fresh fleet of datacenter machines cannot accumulate quickly. A long-time freelancer with one well-tenured peer carries meaningful weight against a freshly-spun-up datacenter.
- **One-CPU-one-voice in PoUH committees.** Block production and validator-set selection (RFC-0004 §Layer 2) are by attested identity, but committee selection via VRF is uniform-random in the attested population — a datacenter's 20 machines have 20 chances of selection out of N, not a 20× weighted vote within a committee.
- **The credible-fork-threat (the bottom of the stack).** Any decision the network produces that the broader contributor body finds illegitimate can be forked away from. The right to fork is the right to dissent, regardless of how the votes counted.

The honest framing: ANTS is a network where compute resources translate into a measurable amount of voice. We minimise the proportionality below 1:1 where the architecture lets us (tenure, fork-choice saturation, the social escape hatch) but do not eliminate it.

## How decisions are made

ANTS borrows its decision process from the IETF and from Linux: **rough consensus, running code, and a final tiebreaker**.

1. **Proposal.** A change to a canonical artefact (RFC, reference client behaviour, brand asset, repository structure) is proposed as a draft document or a pull request. The proposal explains what changes, why, and what it costs.
2. **Discussion.** Open, public, time-bounded (typically one to three weeks). Comments on the PR, threads in Matrix, possibly working group sessions.
3. **Rough consensus.** If, after discussion, there is broad alignment among active contributors and no substantive unresolved objection, the change moves to merge consideration.
4. **Merge decision.** The relevant subsystem maintainer pre-approves; the BDFL performs the merge. In the absence of a subsystem maintainer, the BDFL merges directly.
5. **Conflict resolution.** When consensus does not emerge — when two reasoned positions persist and neither side concedes — the BDFL decides. The decision is recorded with explicit reasoning, so that future contributors understand the precedent.

The BDFL is encouraged to defer to subsystem maintainers and to working groups, and to overrule them only when a decision genuinely cannot be reached without unilateral action. The cost of unilateral decisions accumulates over time; the BDFL is responsible for keeping this cost low.

---

## Code of Conduct and enforcement

The project adopts the Contributor Covenant 2.1, available as `CODE_OF_CONDUCT.md`.

Routine enforcement is the responsibility of a panel of **two to three community moderators**, named by the BDFL but operating with day-to-day autonomy. Moderators handle Matrix room moderation, GitHub issue triage, and first-line response to ordinary Code of Conduct incidents (warnings, temporary mutes, content removal, time-bounded participation suspensions). Decisions are recorded in a published incident log. A moderator must recuse herself from any case where she has a direct stake; the remaining moderator(s) then decide, or escalate if recusal leaves the panel below quorum.

This design exists for two reasons. First, **single-point dependency on the BDFL is fragile**: a single overworked or out-of-pocket maintainer is a poor first responder, and turning every incident into a BDFL decision will quietly produce either a backlogged queue or an unaccountable rubber-stamp. Second, **community legitimacy of enforcement matters**: moderators drawn from the contributor body bring their own judgment and standing, and the panel is the part of the project most visibly governed by people other than the BDFL during the v0.x period.

Serious incidents — accusations of harassment, doxxing, coordinated bad-faith activity, or any incident the moderator panel cannot resolve by consensus — escalate to the BDFL. The BDFL decides with consultation of subsystem maintainers and the moderator panel. Outcomes are recorded transparently in the incident log, with the privacy of affected parties protected.

The BDFL retains a final-veto authority over moderator decisions during the v0.x period. The veto is intended for use only when a moderator panel decision is itself manifestly out of line with the Code of Conduct, and each invocation is recorded with explicit reasoning. Repeated invocation against the moderator panel's reasoned decisions is itself grounds for reconsidering the BDFL role — the same principle stated under §"The BDFL Model" applies here.

After v1.0, routine enforcement remains panel-based; the TSC takes the BDFL's escalation role (with the same two-thirds threshold the TSC uses for canonical artefacts), per the post-v1.0 governance arrangements specified in [`spec/RFC-0007-governance.md`](./spec/RFC-0007-governance.md).

---

## When BDFL ends: the sunset for v1.0

This governance model exists for one reason: ANTS at v0.x is unfinished, the design is in flux, and a single decisive maintainer is the fastest path to coherence. Once the protocol is *finished*, that justification disappears, and the model must change.

**ANTS reaches v1.0 — and BDFL governance ends — when *all five* of the following conditions are met**:

1. **Specification stability.** RFCs 0001 through 0006 (Barter, Cache, Verification, Reputation, Identity, Payment Terms) are merged, have been free of breaking changes for at least six consecutive months, and have each survived at least one independent technical review.
2. **Reference implementation maturity.** The reference client has passed at least one external security audit by a reputable party, has been in production use by at least one non-trivial party for six months, and has at least three subsystem maintainers other than the BDFL with sustained merge activity.
3. **Network scale.** At least one thousand independently operated peers have participated in the network in the trailing thirty-day period, distributed across at least twenty jurisdictions.
4. **Implementation diversity.** At least three independent client implementations exist that interoperate with the reference client and pass a standard conformance test suite.
5. **Contributor density.** At least fifteen active contributors with merged work in the trailing twelve months, of whom at least five have served as subsystem maintainers for at least six months.

When all five conditions are met, the BDFL declares v1.0 and convenes the Technical Steering Committee transition.

If any of these conditions take five years or more to meet, the BDFL is expected to reconsider whether the model itself is failing, and to publish a candid assessment.

---

## After v1.0: the Technical Steering Committee

Upon v1.0, ANTS transitions to governance by a Technical Steering Committee (TSC).

- **Composition.** Seven seats. Three are filled by subsystem maintainers selected by their respective subsystem communities. Three are filled by contributors elected at large by the contributor body. One is the BDFL, serving as transitional chair for a fixed twenty-four-month term.
- **Decision-making.** Simple majority on routine matters. Two-thirds majority on changes to canonical artefacts. Unanimous on changes to this governance document.
- **Term.** Seats rotate every two years, with staggered renewal so that no more than four seats turn over in a single cycle.
- **Chair after the transition.** Elected by the TSC from among its members, for a one-year term, renewable once.

The transition from BDFL to TSC is a single, irreversible event. The former BDFL retains contributor status and may serve future TSC terms by election like any other contributor, but holds no residual special authority.

A separate document, `RFC-0007 — Post-v1.0 Governance`, will specify the TSC mechanics in detail. That RFC must be drafted, discussed, and merged *before* v1.0 is declared. It is the only governance artefact whose merge is performed by the *outgoing* BDFL but whose ratification is a formal community decision.

---

## How this document changes

During the BDFL period, this document may be amended by the BDFL after public discussion of at least three weeks. Amendments are recorded in `CHANGELOG.md`.

After v1.0, this document may be amended only by unanimous TSC vote following a sixty-day public comment period.

The principle behind this is simple: governance documents should be easy to discuss and hard to change in haste. A document that can be changed by a single decisive action is a document that did not need to exist.

---

## In plain language

For anyone arriving without time to read all of the above, here is the short version, with no qualifications:

> Salvatore Calcerano is the maintainer of ANTS for as long as it takes to reach v1.0. He has final say on what enters the canonical specification and the reference implementation. He has no power over forks, over peer behaviour on the network, or over independent operators. He intends to step out of this role into a transitional chair seat as soon as the conditions for v1.0 are met. Anyone may contribute, anyone may dissent, anyone may fork. Decisions are made in public.

---

*This document is released into the public domain under [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/). It is intentionally short of legalese and intentionally explicit about its own limits.*

*Drafted at the founding of the project, May 2026.*
