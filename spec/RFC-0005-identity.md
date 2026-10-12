# RFC-0005 — Identity

**Status:** Draft · v0.3 (early — substantial revision expected)
**Topic:** How a peer proves it is a unique physical machine, without naming who owns it.
**Audience:** You, if you have ever wondered why every previous decentralised AI project either had a token or had a Sybil problem.
**Depends on:** trusted hardware that exists today.

---

## What this document specifies

Every other RFC in the ANTS protocol assumes one thing: that we can tell whether a peer is *one* peer or *many*. The community ledger assigns NCS to peer identities; the reputation chain weighs each attested CPU as one validator vote; the cache restricts write rates by attested identity. All of this collapses if anyone can manufacture peer identities cheaply.

This RFC specifies the identity layer that makes peer-counting cost real money — not via a financial deposit (which would reintroduce plutocracy), but via the cost of unique physical hardware.

The principle: **one attested CPU, one peer identity**. Not one human. Not one company. One CPU. Anyone with multiple CPUs can run multiple peers, paying real hardware costs for each. There is no path to creating thousands of peers cheaply, because there is no path to creating thousands of CPUs cheaply.

---

## Why hardware identity

The Sybil problem in peer-to-peer networks is the problem of distinguishing one entity from many. Every previous solution has had a serious drawback:

- **No defence**: attackers create as many peers as they want. (Most early P2P networks, BitTorrent, Tor, IPFS at the routing layer.)
- **CAPTCHAs / proof-of-humanity**: rejected by us because we want peers, not necessarily humans. A datacentre rack should be welcome; a single human running 47 nodes might be problematic.
- **KYC / real-name registration**: rejected because we refuse to identify our users (Manifesto Thesis 14). Tracking who owns each peer is exactly what we are designing against.
- **Stake-based identity**: works against Sybil, but is plutocratic — those with money have more identities. Rejected by Manifesto Thesis 15.
- **Proof-of-Work identity**: works, but wasteful and biased toward whoever has cheap electricity. Rejected.

**Hardware attestation** is the remaining option. A peer proves it is running on a unique, vendor-attested CPU using a Trusted Execution Environment (TEE) that signs its identity with a key tied to physical silicon. The cost of a peer identity is therefore the cost of a CPU with attestation capability — currently around $300–$3000 depending on tier. That cost is non-zero, non-trivially bypassable, and not concentrated in any single market segment.

---

## The TEE landscape

ANTS accepts attestation from any of these TEE families:

- **Intel SGX** (legacy) and **Intel TDX** (current): x86 server-class confidential compute. Mature, widely deployed in cloud datacentres, well-audited.
- **AMD SEV-SNP**: x86 server-class confidential compute, AMD's equivalent of TDX. Slightly different threat model; equivalent for our purposes.
- **ARM TrustZone** (for ARM Cortex-A series) and **ARM CCA** (Confidential Compute Architecture, for newer Arm v9 systems): mobile and edge-class TEEs. Lower per-device performance but vastly more devices.
- **Apple Secure Enclave**: the TEE in every iPhone, iPad, and Apple Silicon Mac since 2017. Closed-source attestation chain (Apple-signed) but cryptographically robust.
- **Qualcomm QSEE / Hexagon DSP secure mode**: Android flagship phones. Reaches into the Android ecosystem.
- **MediaTek APU secure mode**: emerging, covers many Android phones outside the Qualcomm tier.

ANTS does **not** require a specific TEE family. A peer running on any of the above can attest its identity and join. The protocol specifies the attestation format (a uniform wrapper around vendor-specific attestation reports) and the verification path (the protocol fetches the relevant vendor's attestation service to confirm signatures).

This deliberate multi-vendor stance does two things at once: it broadens the device population (RFC-0001's tier system depends on it), and it diversifies the trust base (no single vendor compromise can take down the protocol).

---

## The attestation flow

When a peer joins the network:

1. The peer generates a long-term ANTS identity keypair *inside* its TEE. The private key never leaves the secure enclave.
2. The peer requests a vendor attestation report binding the ANTS public key to its TEE's hardware identity.
3. The peer submits the attestation to the network: the report contains the ANTS public key plus a signature chain rooted in the vendor's hardware attestation key.
4. Any other peer can verify the attestation by checking the signature chain against the vendor's published attestation service.
5. Once verified, the new peer is recognised as a unique identity. Its ANTS public key is its peer identity for all future interactions.

The attestation report is **not stored on the reputation chain**. It is stored by the peer itself, presented on demand to other peers during handshake. Verification is recomputed locally per interaction. (This avoids inflating the chain with attestation records that don't need global consensus.)

The bytes of that presentation — the ATTEST_REQ / ATTEST_RESP exchange, the per-vendor evidence layouts, and the `report_data` binding that ties the evidence to the connection's peer-id and to its `issued_at` — are pinned in RFC-0008 §11.12.

### v1 admission posture: verify-and-tag

*Added in v0.3.* Attestation *verification* is implemented for Intel TDX and AMD SEV-SNP; attestation *generation* needs real confidential-compute hardware and lands separately. Until it does, no peer can present evidence, and a handshake gate that rejected unattested peers would admit no one. The v1 gate therefore runs in **verify-and-tag** mode:

- a peer that answers ATTEST_RESP with an empty body is admitted as **unattested**;
- a peer whose presentation verifies (RFC-0008 §11.12) is tagged **attested** for that connection;
- a peer whose presentation is **rejected** — evidence that fails the chain, signature or binding check, or a malformed answer (RFC-0008 §11.12) — is disconnected: presenting false evidence is worse than presenting none;
- a peer presenting a vendor the verifier does not support yet, or genuine evidence that is no longer fresh, is **not attested** but stays connected — it presented nothing false, and must refresh before it counts.

Every context that requires attestation — committee selection from the attested population, trustee key rotation (RFC-0010), high-stakes bond admission (RFC-0004 §Bonds) — treats an unattested peer as not attested; nothing relaxes. A network with no attested peers is therefore a **trusted-seed testnet**, not the trustless mainnet. Moving to **hard-gate** mode (reject unattested peers at the handshake) is a protocol-version event announced through the governance process, not a local configuration flag: a network where some peers hard-gate and others do not partitions along the flag.

Attestation reports have **expiration windows** (typically a few weeks to a few months, depending on vendor). When a report expires, the peer must obtain a fresh one. This handles two problems: (a) a stolen TEE that has been remotely repaired or replaced gets a new attestation; (b) a compromised vendor key can be retired by waiting for old attestations to expire.

### Attestation freshness window

*Added in v0.1 (Round 4 amendment) to close the circular reference
raised by the multi-persona Round 2 review (TEE-security persona):
RFC-0010 v0.2 §Trustee key rotation requires "fresh, valid TEE
attestation" but RFC-0005 did not define "fresh."*

A TEE attestation is considered **fresh** for protocol-acceptance
purposes if it was issued by the vendor's attestation service within
the **`ATTESTATION_FRESHNESS_WINDOW`** preceding its presentation. The
default window is **30 days**, calibratable per RFC-0008 §7. The
window is bounded below by the vendor's typical attestation re-issue
cadence (Intel TDX, AMD SEV-SNP, ARM CCA all issue on demand at near-zero
cost; Apple Secure Enclave is less flexible) and bounded above by the
acceptable lag for revocation propagation (longer windows mean
compromised TEE keys remain usable for longer).

The freshness check is a **wall-clock comparison against the
attestation's signed `issued_at` timestamp**, with a tolerance of ±5
minutes for clock skew between the verifier and the vendor's
attestation service. (`issued_at` is signed by being committed in the
evidence's `report_data`, RFC-0008 §11.12; an `issued_at` more than
`ATTESTATION_CLOCK_SKEW` = 5 minutes in the verifier's future is
rejected.) A peer presenting an attestation issued more than
`ATTESTATION_FRESHNESS_WINDOW` ago is treated as not-attested and must
refresh before further interaction.

The window applies uniformly to every context where attestation is
required: the standard peer-handshake flow above, trustee key-rotation
announcements (RFC-0010 §Trustee key rotation), bond admission for
high-stakes acts (RFC-0004 §Bonds), and committee role assumption
(RFC-0004 §Layer 2). "Fresh" means the same thing everywhere it
appears in the corpus.

**Why 30 days, not stricter.** A stricter window (7 days) would force
peers behind intermittent connectivity to re-attest more often than
their network access supports — punishing the home-user peer the
protocol explicitly welcomes. A looser window (90 days) would extend
the post-vendor-compromise window during which old attestations remain
usable. Thirty days is the rough midpoint that matches typical
vendor revocation-propagation latency in 2026 and the cadence at which
honest operators can reliably refresh without operational friction.
The value is calibratable on b2 evidence; the *mechanism* — wall-clock
freshness with a defined window — is the contribution this section
makes.

---

## Multi-vendor reputation weighting

Even though the protocol accepts attestation from any major TEE vendor, it does not treat them as fully fungible. A peer's reputation weight is modulated by **attestation diversity**:

- A peer with a single-vendor attestation has reputation weight 1.0.
- A peer that supplements its primary attestation with a second-vendor attestation (e.g., an Intel TDX peer also running its identity binding through a co-located ARM device) has weight 1.2.
- A peer with three independent vendor attestations has weight 1.4.

This creates a small but real incentive for multi-vendor diversity at the high-value peer tier (datacentres, professional operators). For the median home-user peer with a single device, single-vendor attestation is fine and remains the default.

The system-level consequence: the *most influential* peers in the network — those validating blocks, hosting popular cache shards, serving expensive Tier 3 verifications — are statistically more likely to have multi-vendor backing. A coordinated compromise of any single vendor weakens the network proportionally but does not collapse it.

---

## Sybil economics

The cost of attempting to control a meaningful fraction of the network via fake hardware identities:

| Network size | 10% control needs | Yearly cost (cloud rental) |
|---|---|---|
| 1,000 peers | 100 attested CPUs | ~$70,000 |
| 10,000 peers | 1,000 attested CPUs | ~$700,000 |
| 100,000 peers | 10,000 attested CPUs | ~$7,000,000 |
| 1,000,000 peers | 100,000 attested CPUs | ~$70,000,000 |

These are cloud-rental costs from major confidential compute providers (Azure Confidential, AWS Nitro Enclaves, GCP Confidential VMs) at 2026 prices for low-end confidential compute instances. Purchasing physical hardware is comparable.

At all but the very smallest network sizes, the cost of Sybil attack exceeds any plausible economic gain from such an attack. The network is, by economic construction, defended against fake identities.

The cost is **not high enough to defend against nation-state attackers** at small network sizes. A government willing to spend $10M to corrupt the protocol when the network has only 1,000 peers could plausibly succeed. This is one of several reasons the protocol's early phase is the most vulnerable, and one of several reasons the manifesto declares (Thesis 16) that ANTS is stewarded by a foundation, not a company — to make the foundation a credible coordinator for emergency response during the vulnerable period.

### Price erosion: the snapshot is not a permanent floor

The table above is a 2026 snapshot, not a permanent floor. Confidential
compute pricing has followed a clear downward trajectory since its
mainstream availability around 2020: roughly **15–25% real-terms
year-over-year decline** as the underlying silicon moves from premium
SKU into commodity server fleets and as a second wave of providers
competes with the original three hyperscalers. There is no Moore's-law
guarantee here — confidential compute is constrained by attested silicon
supply, not transistor count — but the trend is real and the protocol
must plan for it.

A useful projection, taking 20%/yr as the central estimate, with the
honest caveat that the trend can flatten or reverse if (a) attested
silicon supply tightens, (b) regulatory mandates push cost up, or (c)
TEE vendors price for differentiation rather than volume:

| Year | 10% control of a 100,000-peer network | Equivalent in 2026 dollars |
|---|---|---|
| 2026 | ~$7.0M | $7.0M (baseline) |
| 2028 | ~$4.5M | (~36% cheaper than 2026 in real terms) |
| 2030 | ~$2.9M | (~59% cheaper) |
| 2032 | ~$1.9M | (~73% cheaper) |
| 2035 | ~$1.0M | (~86% cheaper) |

The mechanism the protocol relies on as price erodes is **network
growth**. Sybil cost scales linearly with target network size (10% of N
peers costs 0.1·N·c_TEE, where c_TEE is the per-peer rental). As long
as N grows faster than c_TEE falls, the absolute Sybil cost rises.
RFC-0010's Phase 3 (100–999 peers) and Phase 4 (1,000+) are the windows
in which this race is decided.

**The honest framing.** Hardware-attested identity is a **decade-class
Sybil deterrent, not a perpetual one**. The half-life of the deterrent's
absolute strength under price erosion is roughly 4 years at 20%/yr. The
protocol's long-term defence against this erosion is not "TEE will stay
expensive forever" — it is the combination of (a) network growth
outpacing price decline, (b) the open-hardware migration of §"The
long-term open-hardware path" reducing dependence on vendor pricing
power entirely, and (c) the (A, T, κ) spine in RFC-0004 making mere
identity-creation insufficient — the attacker must also amortize each
identity with sustained honest contribution, which is what we wanted
all along.

A perpetual hardware-only Sybil deterrent does not exist. The honest
answer is the stack of three defences above, not the table.

### The 2028–2030 vulnerability window

*Added Round 4 EDGE to address the concern raised by the multi-persona
Round 2 review (TEE-security persona): the three long-term-wall defences
are cumulative in the corpus but additive in effectiveness only if all
three hold.*

The three defences above (network growth, open-hardware migration,
(A, T, κ) amortisation) are designed to overlap so that none is
load-bearing alone. They do overlap — *in steady state*. There is a
transient window where each is structurally weak at the same time, and
the corpus should name it rather than let it sit as an unstated risk:

- **Network growth** is at its weakest before RFC-0010 Phase 3 (N ≥ 100
  attested peers) is reached. Pre-Phase-3, the 10%-control cost of the
  network is at most a few × the per-peer cost — comfortably within
  state-actor budgets and within a determined commercial adversary's
  budget.
- **Open-hardware TEE migration** is realistically 2030–2032 per
  §"The long-term open-hardware path", not 2027–2028 as the v0.1
  (early) draft hoped. Until then, all attestation flows through the
  five named closed-vendor families.
- **(A, T, κ) amortisation** is calibrated by b2 against a running
  testnet. The testnet does not yet exist; the reference client is on
  the 24–36 month critical path of IMPLEMENTATION.md; b2 measurement
  cannot begin until then. The κ-rate-cap that makes the amortisation
  work is consequently still a default parameter, not an empirically
  tuned one.

The window where all three defences are *simultaneously weak* is
therefore approximately **2028–2030**: after the reference client is
likely to be shipping but before either open-hardware TEE has matured
or the network has reached Phase 3 scale. In this window, the
protocol's Sybil resistance rests largely on the cost-asymmetry wall
(§Sybil economics) and on the (A, T, κ) spine running with default —
not measured — parameters.

This is not a defect we will eliminate by writing differently. It is
the *expected* vulnerability profile of a permissionless protocol
bootstrapping from zero. The honest mitigations during this window are
operational, not architectural:

1. **The foundation per Manifesto Thesis 16** acts as a credible
   coordinator for emergency response during the vulnerable phase —
   exactly the role for which the foundation exists. The foundation is
   not the network's authority; it is the network's *first responder*
   while the network is too small to defend itself.
2. **GOVERNANCE.md sunset checkpoints** (the 6-month and 24-month
   reviews) include explicit attention to the vulnerability window: if
   Phase 3 has not been reached at the 24-month checkpoint and the
   reference client is not yet shipping b2 data, the BDFL publishes a
   candid assessment per the "5-year clause."
3. **The credible-fork-threat** remains the irreducible coordination
   floor: a network that begins to fail in 2028–2030 can still be
   forked by its honest contributors to a recovered state, with all
   the well-specified mechanics of RFC-0004 §Fork-recovery applied.

The window will pass either by network growth reaching scale, by
open-hardware acceptance arriving, by b2 calibration converging, or
(in the failure case) by the candid assessment that the protocol's
particular design needs reconsidering. Naming the window now lets the
project measure progress against it rather than discover it as a
surprise.

---

## Anti-eclipse peer sampling

RFC-0004's entire Layer-1 security reduces to one structural property: a
monotone, self-authenticating fault proof needs **one honest gossip path** from
detector to victim within `T_prop` — not an honest majority (RFC-0004 §"The
structural win"). That property is *conditional*. It holds only while an
adversary cannot **vertex-cut the honest subgraph around a victim** — cannot fill
that victim's gossip view entirely with colluding peers. Filling a view is an
**eclipse**; an eclipsed peer never receives the slash that would retire a
colluder, and `T_prop < T_beacon` (RFC-0004 §"The binding cross-layer
constraint") silently becomes vacuous for them. RFC-0004 names this as RFC-0005's
problem and stops. This section fixes the *mechanism* that makes an eclipse
expensive — and is honest about what it does **not** prove.

### The view is sampled, not curated

The gossip overlay (Component #6) deliberately does not choose its own
neighbours: its view is a set the surrounding node populates. The questions are
*who* populates it and *how*. Two structural choices.

**Placement — the DHT owns sampling.** The Sybil-resistant selection is a
sampling primitive of the DHT (Component #5) — not inside the gossip engine, and
not in per-deployment integration glue. The DHT already holds the peer knowledge
(a routing table keyed by XOR distance, plus iterative lookup), so it is the
natural and *single* place to make eclipse-resistant selection testable.
Embedding selection in the gossip engine would couple two network components and
make the dissemination logic untestable without a live DHT; leaving it to glue
would scatter the security-critical decision across every reimplementation,
untested. The gossip engine stays substrate-agnostic — it takes peers via
add/remove — and a conforming node wires the DHT's sample into the gossip view,
refreshing it on a timer.

**Selection — diversity from three independent axes.** No single source of peers
is safe (a poisoned routing table eclipses by itself). The sample combines:

1. **Bucket-diverse routing-table draw.** Peers spread across XOR k-buckets, so
   the view spans the keyspace rather than clustering where an attacker can
   cheaply position identities.
2. **Random-target probes.** Periodic lookups toward random keys pull in peers
   from regions of the keyspace an adversary cannot pre-position around a chosen
   victim — the victim's future probe targets are unpredictable.
3. **Rotation.** A fraction of the view is replaced each refresh, so a static
   eclipse cannot lock in: the adversary must hold a majority of the view *and*
   keep re-winning it against churn.

A fourth axis — **attestation-weighted selection**, preferring and diversifying
across distinct *attested* hardware identities (this RFC's machinery) — is the
strongest defence and is **deferred**. It is the part that turns "diverse peers"
into "diverse peers an adversary cannot cheaply manufacture", and it depends on
the attestation flow above being deployed and on §"Multi-vendor reputation
weighting". Until it lands, axes 1–3 are defence in depth, not a Sybil-resistance
guarantee.

### The recursion, stated plainly

Anti-eclipse wants Sybil-resistant peer selection; Sybil-resistance is this RFC,
which leans on the very Layer-1 reputation that anti-eclipse protects. The loop
is real and is not eliminated here. It is **bounded** by the one-honest-path
asymmetry: a victim needs *one* honest peer reachable in its view, not an honest
majority, so even the unauthenticated diversity of axes 1–3 raises the attacker's
required share of a victim's view far above what a consensus system tolerates.
Attestation-weighting (axis 4) tightens the bound further by pricing each view
slot at the cost of an attested identity. The loop closes to a *cost*, not to
*impossibility*.

### Parameters (testnet-calibrated)

The view is governed by its size, the eager fanout `f` (RFC-0008), the rotation
fraction, the random-probe count per refresh, and the refresh interval. All feed
`T_prop ≈ c·Δ·log N` and are therefore jointly constrained with
`T_prop < T_beacon`; like RFC-0003's `e` and RFC-0001's `w`, they are **measured
on a testnet under partially-adversarial topology**, not chosen at the desk. They
are named here so the testnet measures against them rather than discovering them
as a surprise.

### What this does not prove

This section makes eclipse *expensive*; it does not make it *impossible*, and it
does not deliver a provable bound on eclipse probability under a stated
adversary. Axes 1–3 are heuristics; the rigorous guarantee waits on
attestation-weighted selection (axis 4) **and** a formal eclipse-probability
model. Concretely: **RFC-0004's one-honest-path assumption remains an
assumption** — now narrowed and made costly, not discharged — and must not be
reported in RFC-0004 as resolved. That is the load-bearing residual of this
section, and the honest one: ANTS's Layer-1 liveness rests, at the bottom, on an
anti-eclipse assumption this design makes expensive to violate and that a later
attestation-weighted, formally-modelled revision must make precise.

---

## The long-term open-hardware path

A protocol that depends permanently on Intel, AMD, ARM, and Apple is dependent on four specific corporations. This is acceptable in 2026, where these vendors have the only mature TEEs. It is not acceptable as a permanent state.

The protocol's identity layer commits to migrating, over the next decade, toward open-hardware TEE roots. Specifically:

- **RISC-V Keystone** is a research effort to build a fully open-hardware TEE on RISC-V cores. Not production-ready in 2026, but advancing.
- **OpenTitan** (Google + lowRISC + Western Digital + others) is an open-source silicon root-of-trust. Some commercial implementations exist; broader adoption pending.
- **MIT's open secure enclave research** and several academic projects are exploring formally-verified TEE architectures.

The protocol commits to:

1. Accept open-hardware TEE attestations *as soon as* they are
   production-grade. Realistic horizon: **2030–2032** for first
   production-grade Keystone or OpenTitan acceptance. The earlier
   2027–2028 estimate in v0.1 (early) of this RFC was optimistic; open
   silicon, audit, supply chain, and ecosystem maturity together take
   longer than the research roadmap implies. We say so honestly here
   rather than carry a date that will quietly slip.
2. Weight open-hardware attestations *higher* than closed-vendor
   attestations in reputation calculation, as a deliberate incentive
   for migration.
3. By v2.0 (long after v1.0), the protocol intends to require *at least
   one* open-hardware attestation for validator participation —
   eliminating the dependence on closed corporate roots-of-trust at the
   most security-critical layer.

This is a multi-year migration measured in **half-decade scale, not
2–3-year scale**. It is included in this RFC because the alternative —
pretending closed hardware is permanently acceptable — would be
dishonest. Pretending the migration is around the corner would also be
dishonest, and we now say that too.

---

## Privacy

The hardware attestation contains:
- A vendor-signed cryptographic identifier tied to the physical CPU.
- The ANTS public key generated inside the TEE.
- Sometimes optional metadata about the TEE state (firmware version, security patches applied).

The attestation does **not** contain:
- The owner's real name, email, phone, or any human-identifying information.
- The TEE's physical location, except as inferable from network routing.
- The peer's IP address as part of the cryptographic identity.

Hardware-attested peer identities are unlinkable to real-world owners by inspection of the chain or the network. They are *linkable to themselves over time* — the same CPU appears repeatedly — which is exactly what we want.

Practically, peers may *choose* to publish associations between their hardware identity and a human identity (e.g., a public researcher operating under their real name). This is voluntary, off-protocol, and the protocol does not endorse or facilitate it.

---

## What we have not figured out yet

- **Attestation report formats vary by vendor**, and we need a uniform wrapper. The Confidential Computing Consortium's Attestation work is the candidate; specifications pending.
- **Revocation propagation.** When a hardware identity is slashed at the reputation chain, all other peers must learn this and stop accepting that identity's attestation. Mechanism: gossip + chain consultation. Latency tolerance: needs spec.
- **Recovery from TEE firmware updates.** A peer's TEE may receive a firmware update that changes its attestation signature without the underlying hardware identity changing. The protocol needs to handle continuity of identity across legitimate firmware changes while detecting illegitimate ones.
- **Air-gapped attestation refresh.** A peer that loses internet temporarily and cannot refresh its attestation — does it lose its identity? Should there be a grace period?
- **Cross-platform identity binding for one human.** If a single user has both a laptop and a phone, both attested separately, they have two ANTS identities. Should the protocol provide a way to optionally link them as "controlled by the same human"? Probably yes for UX reasons, but voluntary and unverifiable by the protocol.
- **Apple Secure Enclave's closed attestation chain.** Apple's attestation roots are not public the way Intel's and AMD's are. We accept their attestation but cannot independently audit the chain. This is an acceptable trade-off in 2026; it becomes less acceptable as the protocol scales.
- **Vendor refusal to attest specific peers.** What if Intel, under government pressure, refuses to issue attestations to peers in jurisdiction X? The protocol cannot prevent this. Mitigation: multi-vendor diversity ensures that no single vendor's refusal can keep peers out of the network entirely.

---

## Adversarial considerations

**Vendor compromise.** A vendor's attestation signing key is compromised. The attacker can mint unlimited fake identities. Mitigations: (a) multi-vendor weighting means single-vendor minting attacks are detected as anomalies (sudden spike in single-vendor identities); (b) the vendor publishes a key rotation, old attestations are invalidated, peers re-attest with the new key; (c) in extreme cases, the affected vendor is temporarily downweighted by network coordination.

**Vendor revocation propagation timing** *(added Round 4 EDGE).* When a vendor publishes a revocation (TCB bump, key rotation, or specific batch invalidation), the protocol bounds the window during which old attestations remain accepted by `VENDOR_REVOCATION_PROPAGATION_BOUND` (RFC-0008 §7, default **72 hours**). The mechanism: revocations are propagated as L1 CRDT objects (same gossip channel as fault proofs), each carrying the vendor's signed revocation manifest. After 72 hours from a revocation's first observation by any honest peer, peers reject attestations matching the revoked identifier regardless of whether they have personally seen the revocation. The bound is calibratable on b2 evidence; 72 hours is the rough fit to typical vendor disclosure-to-network-propagation latency in 2026 and is short enough that a post-disclosure attack window is operationally costly but long enough that honest operators in poorly-connected jurisdictions have time to refresh. A peer presenting a revoked attestation after the bound is treated identically to a peer presenting a stale attestation per §Attestation freshness window.

**TEE vulnerability discovery.** A new side-channel attack (similar to historical examples like Spectre/Meltdown) allows extracting attestation keys from a TEE. Mitigations: most attacks require physical or co-located access, limiting scale. Vendors release firmware patches; old vulnerable peers are flagged.

**Hardware re-attestation farming.** An attacker buys 1000 TEE-capable devices once and re-uses them indefinitely. Defence: this is not really an attack — the attacker has 1000 identities for the cost of 1000 devices, which is the expected pricing. Nothing to mitigate.

**Stolen TEE.** A laptop is stolen, the thief tries to use its ANTS identity. Defence: the identity is bound to the TEE, which the thief now has. But the original owner can publish a revocation event (signed by their own backup key, which the protocol provides at identity creation time). The stolen identity is then untrusted.

---

## Historical TEE vulnerabilities and the protocol's response posture

*Added Round 4 EDGE to address the concern raised by the multi-persona
Round 2 review (TEE-security persona): the corpus depends on TEE
attestation and does not catalogue the historical vulnerability record
of that hardware, leaving each new disclosure as a future emergency.*

TEEs have been broken before. They will be broken again. A protocol
that pretends otherwise is a protocol that handles its first CVE as a
surprise. This section catalogues the recurring failure modes so the
protocol's response posture is specified before, not during, the next
disclosure.

### A non-exhaustive catalogue

| Year | Name | Affected family | Class | Vendor response |
|---|---|---|---|---|
| 2018 | Spectre / Meltdown / Foreshadow | Intel SGX, all x86 | Speculative-execution side channel | Microcode + OS mitigations; SGX TCB bump |
| 2019 | Plundervolt | Intel SGX | Voltage-fault injection | Microcode disable of undervolting from untrusted code |
| 2020 | SGAxe | Intel SGX | Cache side-channel | TCB recovery + attestation re-issue |
| 2021 | ÆPIC Leak | Intel SGX | Architectural register leakage | Microcode + TCB bump |
| 2023 | Downfall | Intel SGX, AVX | Gather-instruction microarchitectural leak | Microcode patch + measurable performance impact |
| 2023 | Inception (Phantom) | AMD SEV-SNP | Branch-prediction-based control-flow hijack | AGESA firmware update |
| 2023 | CacheWarp | AMD SEV-SNP | Cache-line write rollback | Firmware update |
| 2024 | Heckler | AMD SEV-SNP | Malicious-hypervisor interrupt injection | Firmware + Linux kernel mitigations |
| various | Cross-VM Spectre variants | Cloud-hosted TEEs (any vendor) | Co-tenant side channel | Hypervisor isolation + scheduler mitigations |

The list is illustrative, not complete. Each entry shares a structure:
disclosure → vendor TCB-recovery procedure → attestation re-issue with
a new firmware identifier → old attestations marked stale → time for
peers to refresh.

### The response posture

The protocol's posture in front of a new disclosure is **the same on
day one as on day N**: every attested peer is bound to a specific TEE
firmware identifier carried in its attestation, every attestation is
checked against the freshness window of §Attestation freshness window,
and the vendor's TCB-recovery is propagated as ordinary attestation
re-issue. Three concrete behaviours:

1. **No protocol-version bump is required to absorb a new
   vulnerability.** The mechanism for accepting "the new TCB" and
   rejecting "the old TCB" is built into the freshness-window check
   and the vendor's published revocation. The protocol does not need
   to redeploy code to handle a CVE; peers need to refresh attestations.
2. **The multi-vendor weighting (§Multi-vendor reputation weighting)
   contains the blast radius.** A vulnerability affecting one vendor
   family weakens but does not collapse the network. Peers with
   multi-vendor attestation continue at near-full weight; single-vendor
   peers on the affected family drop to zero weight until they refresh
   to a patched TCB or supplement with another vendor.
3. **The vendor's revocation timing is bounded.** §"Vendor revocation
   propagation" (below) makes the revocation-propagation latency
   explicit, so the post-disclosure window of "old attestations still
   accepted" is bounded, not undefined.

The honest framing: the protocol cannot prevent TEE vulnerabilities
from being discovered. It can ensure that the discovery is absorbed by
the same primitives the protocol already uses, in a bounded time, with
graceful degradation rather than catastrophic failure. That posture is
the contribution this section makes — not a claim that TEEs are safe,
but a specification of what happens when they are not.

### What this does not solve

A **zero-day exploited in the wild before disclosure** is the case
this section cannot defend against. An attacker with a working
side-channel against (say) AMD SEV-SNP, used quietly to mint Sybil
identities, leaves no observable trace until the vulnerability is
publicly disclosed. The multi-vendor weighting limits the damage —
single-vendor anomalies are detectable as anomalies — but does not
prevent the attack window.

This is part of the irreducible residual handed to the
credible-fork-threat. A network that learns *post hoc* that 30% of its
attested identities were minted via an undisclosed SEV-SNP exploit can,
in principle, reissue identities on the basis of the disclosed-and-patched
TCB and fork away from the compromised state. The mechanism is the
same `Σ T_eff` fork-choice + social-Schelling fallback that handles
every other state-actor-residual scenario.

---

## How to contribute to this document

Open an issue on `Ants-Community/ants` with the tag `rfc-0005`. Particularly valuable: cryptographic analysis of the multi-vendor weighting scheme, empirical data on cloud TEE costs, alternative Sybil-resistance mechanisms we have not considered.

The document is CC0.

---

*Drafted by the ANTS founding contributors, May 2026.
One CPU, one voice. The hardware is the identity.*
