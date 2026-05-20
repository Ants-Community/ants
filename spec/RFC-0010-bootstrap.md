# RFC-0010 — Bootstrap Sequence

**Status:** Draft · v0.4
**Topic:** What the first peer does when it runs the reference client, and how the network evolves from one attested peer to a self-secured colony.
**Audience:** You, if you are about to implement the bootstrap of the reference client.
**Depends on:** [GOVERNANCE.md](../GOVERNANCE.md), [RFC-0001](./RFC-0001-community-economy.md), [RFC-0002](./RFC-0002-semantic-cache.md), [RFC-0003](./RFC-0003-verification.md), [RFC-0004](./RFC-0004-reputation-pouh.md), [RFC-0005](./RFC-0005-identity.md), [RFC-0008](./RFC-0008-wire-formats.md), [RFC-0009](./RFC-0009-canonical-numerics.md)

---

## Why bootstrap is hard

Each RFC in the corpus assumes the network already exists. RFC-0001's
choke/unchoke needs peers to choke. RFC-0002's DHT needs peers to route
through. RFC-0003's verification tiers need verifier-population peers.
RFC-0004's PoUH chain needs attested peers for committees. RFC-0005's
identity assumes you can present your attestation to other peers.

None of those things exist at `t = 0`.

The all-in-one reference client (per the GOVERNANCE.md implementation
strategy) must therefore answer a question the design RFCs sidestep: **what
does `main()` do on the first peer?** And then on the second, third, and
hundredth?

This RFC specifies that. It closes BLOCK B4 from the pre-implementation
criticality review. It is the document the reference-client engineers
read first.

---

## Genesis state

Every peer joining the network must agree, bit-for-bit, on a small set of
initial constants — the **genesis state**. Disagreement on any of these is
disagreement about which network you are on; two peers with different
genesis states are on different protocols, not different views of the same
one.

The genesis state is a CBOR-encoded structured object per
[RFC-0008](./RFC-0008-wire-formats.md) §1.1, hashed with BLAKE3 to produce
the **genesis hash** that pins the network identity:

```
GenesisState {
  protocol_version:      "ants-v1",
  protocol_version_hash: BLAKE3 hash of the merged spec corpus at launch,

  embedding_model: {
    id:                  "ants-embed-v1",
    arch:                "bge-m3",
    weights_hash:        BLAKE3(canonical safetensors file),
    tokenizer_hash:      BLAKE3(canonical tokenizer.json),
  },

  trustees: [
    {
      peer_id:           Ed25519 pubkey,
      bls_pubkey:        BLS12-381 G1 pubkey (derived per RFC-0008 §3.4),
      tee_attestation:   opaque vendor blob,
      granted_weight:    initial granted weight (μNCS-equivalent),
    },
    ... N trustees ...
  ],

  trustee_decay_rate:    δ_genesis (per RFC-0004 v0.3 §Fork-recovery),
  trustee_sunset_curve:  G(0)·e^{-δ_genesis · t} schedule,

  drand: {
    network:             "default-drand-mainnet",
    chain_hash:          drand network chain hash,
    initial_round:       integer round at protocol launch + 1,
  },

  bootstrap_dht_seeds: [
    multiaddr_1,         (well-known initial DHT peers)
    multiaddr_2,
    ...
  ],

  initial_constants: {
    (the RFC-0008 §7 Reference Constants table, every value populated)
  },

  launch_timestamp:      ISO-8601 UTC,
  launch_signature:      BLS aggregate over all trustee BLS keys covering
                         BLAKE3(all of the above),
}
```

The **genesis hash** is `BLAKE3.derive_key("ants-v1-genesis", GenesisState
canonical-CBOR encoding)`. Every peer's first action on launch is to verify
this hash against the value baked into the reference client binary. A
mismatch means the peer is being asked to join a different network than the
one the binary was built for; the client refuses to start.

The genesis state is published once, signed by all trustees, and never
amended in-protocol. To change it requires a major protocol-version bump
(RFC-0008 §9) and a coordinated migration — which is to say, a different
network.

---

## The first peer's flow

The reference client's bootstrap sequence, in execution order, for any peer:

```
main():
  1. Load embedded GenesisState; verify genesis hash
  2. Generate or load Ed25519 keypair INSIDE the TEE
  3. Derive BLS12-381 keypair per RFC-0008 §3.4
  4. Obtain a fresh TEE attestation binding the Ed25519 pubkey to hardware
     (RFC-0005: Intel TDX / AMD SEV-SNP / ARM CCA / Apple Secure Enclave / Qualcomm QSEE)
  5. Connect to ≥3 bootstrap_dht_seeds from GenesisState (round-robin, retry)
  6. Announce self in the DHT: peer_id, multiaddrs, supported services,
     attestation public bundle
  7. Sync the L1 CRDT (RFC-0004 Layer 1): receive all currently-known
     fault proofs and bond-admission events
  8. Sync the L2 PoUH chain (RFC-0004 Layer 2): from the genesis block (signed by trustees)
     forward to the current head; verify every committee signature against
     the trustee set at that block height
  9. Initialize the local economic ledger (RFC-0001): empty, ready for the
     first pair-wise interactions
 10. Decide on cache participation: choose a shard region by hash of peer_id
     mod shard_space; start hosting if storage allows
 11. Open service slots per RFC-0001 §choke/unchoke: ready to serve and
     consume
 12. Subscribe to drand for VRF beacon values per RFC-0008 §4.2
 13. Begin participation: route queries, serve cache hits, perform Tier 1
     audits when the beacon selects you as auditor

[ steady state ]
```

Steps 1–4 are local. Steps 5–8 are network-bound and may take seconds to
minutes depending on chain height. Steps 9–13 happen continuously after
that.

**Error handling**: if the genesis hash mismatches (step 1), the client
refuses to start. If TEE attestation fails (step 4), the client refuses to
start. If no bootstrap DHT seeds respond within a timeout (step 5), the
client retries with exponential backoff and warns the operator. The chain
sync (step 8) is mandatory — a peer that has not synced cannot be admitted
to any high-stakes act because it cannot verify other peers' bond-admission
events.

---

## Phase transitions

The network evolves through four phases by total attested-peer population.
Each phase has different capability availability, different security
assumptions, and different governance posture.

### Phase 1 — Tiny network (1 ≤ N < 10)

The phase that lasts the first hours-to-days after launch.

- **PoUH committee size `K`** is bounded by `N` and the bootstrap schedule:
  starts at `K = min(N, 5)`, signed by all online trustees.
- **Signature scheme**: Ed25519 multi-signatures (per RFC-0008 §3.3 v0.2,
  K ≤ 16 uses Ed25519).
- **Block time**: 30 seconds per RFC-0008 §7, but with frequent
  `degraded_seed: true` flags if drand hasn't been observed yet by the few
  online peers.
- **Verification tiers available**: Tier 1 only. Tier 2/3 require enough
  attested peers in the verifier pool to be statistically defensible;
  with N < 10, an audit would have ≥ 30% chance of selecting an
  uninvolved peer, which is acceptable for *some* anti-fraud but not for
  the strict Type-I-error guarantees RFC-0003 promises.
- **Cache**: essentially empty. Cache hits dominated by trustee-produced
  entries during initial seeding.
- **Tenure (`T`) accrual**: capped at `κ` per peer per time as always; no
  peer has accumulated meaningful tenure yet. Verifier-set eligibility
  is **gated by trustee endorsement** during this phase — a non-trustee
  peer cannot serve as a verifier until it has reached the
  `TENURE_FLOOR` constant, which under nominal `κ` takes weeks.
- **Genesis trustee influence**: ~100% by construction.

### Phase 2 — Small network (10 ≤ N < 100)

The first natural phase. Lasts roughly weeks-to-months.

- **PoUH committee `K`** grows to `min(N/2, 16)`. Still Ed25519 multi-sig.
- **Verification tiers available**: Tier 1 fully. Tier 2 becomes possible
  as the auditor pool grows past ~20 attested peers; below that, Tier 2
  audits have non-trivial probability of selecting the producer itself
  as auditor (catastrophic for the no-collusion property), so Tier 2 is
  formally enabled only at `N ≥ 20`. Tier 3 still requires committees of
  3+, which is fine, but a meaningful pool requires ~50+ peers.
- **Cache**: starts to accumulate entries. Specialist credit (RFC-0002
  §"Specialist credit and the cache") begins to function.
- **Tenure**: the first non-trustee peers cross `TENURE_FLOOR`. They
  become eligible to serve as auditors, joining the verifier set.
- **Genesis trustee influence**: starts decaying per the
  `e^{-δ_genesis · t}` curve, but still dominates total weight.
- **Failure mode to watch**: if N stalls below 50 for an extended period,
  the network is *not* self-securing and the genesis trustees retain
  substantive authority indefinitely. GOVERNANCE.md's "if these
  conditions take 5+ years to meet, the BDFL reconsiders" applies.

### Phase 3 — Growing network (100 ≤ N < 1000)

The network reaches scale where it can plausibly defend itself.

- **PoUH committee `K`** grows past the BLS/Ed25519 transition threshold
  (`K = 16` per RFC-0008 §3.3 v0.2). BLS aggregate signatures take over
  for blocks at `K > 16`.
- **All verification tiers available**: Tier 1, Tier 2, Tier 3.
- **Cache**: at scale where the specialist-credit mechanism is producing
  durable yield. The cache hit ratio crosses the threshold (estimated
  60%+ for established domains) where the network's effective compute
  multiplier dominates.
- **Tenure distribution**: many peers have crossed `TENURE_FLOOR`. The
  verifier set is no longer trustee-gated.
- **Genesis trustee influence**: decayed to the percentages defined by
  the sunset curve at this network age (typically 10–25% if launch was
  6–12 months earlier).
- **First plausible fork-recovery scenario**: a contested decision
  (embedding model upgrade per RFC-0002 v0.2, or contested slashing)
  may produce a non-trivial fork attempt. The credible-fork-threat
  becomes tested for the first time.

### Phase 4 — Mature network (N ≥ 1000)

The "self-secured" phase. All security claims of the corpus apply at full
strength.

- **PoUH committee `K = 64`** (RFC-0008 §7 default). BLS aggregation
  throughout.
- **All verification tiers** at full capacity. The Type-I-error and
  Type-II-error guarantees of RFC-0003's e-process hold under realistic
  honest-peer-population assumptions.
- **Cache**: mature. Specialist credit is the dominant economic
  arrangement.
- **Tenure**: well-distributed. The (A, T, κ) spine functions as
  designed across thousands of peers.
- **Genesis trustee influence**: decayed to negligible (single-digit
  percentage at most). The genesis trustee role is essentially
  vestigial; the credible-fork-threat now rests on the broader
  community.
- **Sybil cost** (from RFC-0005): ~$7M/year cloud-TEE rental for 10%
  network control. Effective against most adversaries; remains the
  state-actor wall.

---

## Capability matrix per phase

A table the reference client can consult to enable/disable features:

| Capability | Phase 1 (1-9) | Phase 2 (10-99) | Phase 3 (100-999) | Phase 4 (1000+) |
|---|---|---|---|---|
| Tier 1 attestation | ✓ | ✓ | ✓ | ✓ |
| Tier 1 baseline sampling | trustees only | from N ≥ 20 | ✓ | ✓ |
| Tier 2 cross-check | × | from N ≥ 20 | ✓ | ✓ |
| Tier 3 triangulation | × | from N ≥ 50 | ✓ | ✓ |
| PoUH block production | trustee-signed | K up to 16, Ed25519 multi-sig | K up to 32, BLS | K = 64, BLS |
| `degraded_seed` flag | common | occasional | rare | exceptional |
| Cache serving | bootstrap entries | building | mature | dominant |
| Cache hosting | optional | recommended | recommended | recommended |
| Tenure-gated verifier set | trustees + endorsed | tenure-floor crossers | unrestricted | unrestricted |
| Fork-recovery viable | trustee-coordinated | trustee-coordinated | possible | full social-Schelling |
| High-stakes bonds (RFC-0004 v0.4) | limited (few peers have A) | available | available | available |
| All RFC-0009 canonical numerics | required | required | required | required |
| BGE-M3 embedding pinning | required | required | required | required |

A peer that attempts a Phase 3/4 capability when the network is in Phase 1/2
fails gracefully: the request is declined with reason `phase_too_early` and
the request falls through to whatever lower-phase capability is available.
No misbehaviour is recorded.

---

## The self-securing condition

The network is **self-securing** at network size `N` when the genesis
trustee influence `Φ_genesis(t) < θ` AND the honest-peer-derived tenure
satisfies `W_h ≥ (1 − θ) / θ · W_a` (per RFC-0004 v0.3 §Fork-recovery's
D2 condition).

In practical terms, this means:

```
Φ_genesis(t)  =  G(0) · e^{-δ_genesis · t}  /
                 ( G(0) · e^{-δ_genesis · t}  +  Σ over non-trustee peers of T_i(t) )
```

drops below θ (default θ = 0.5 for honest-majority) when the non-trustee
tenure sum dominates the decaying trustee weight. Under nominal `κ` and
`δ_genesis`, this happens around N ~ 200–500 attested peers active for
~3–6 months.

The exact value is a calibration target — the reference client tracks
`Φ_genesis` continuously and surfaces it in the operator dashboard. Once
it drops below θ, the network is no longer dependent on trustee honesty
for security; the credible-fork-threat is intrinsic.

A network that fails to reach the self-securing condition within a
GOVERNANCE.md-specified time (the 5-year clause) means the bootstrap has
failed and the protocol design needs reconsidering. The honest framing.

---

## The reference launch sequence

This subsection specifies what "v0.1 mainnet launch" concretely is. It is
not aspirational; it is the recipe.

1. **Trustee selection.** Founding contributors named in GOVERNANCE.md and
   their public-key delegates form the initial trustee set. The set size
   is bounded at 7–11 individuals to keep coordination tractable. Each
   trustee generates their identity inside their own TEE; the public keys
   plus attestations are aggregated.

2. **GenesisState assembly.** All values populated, the canonical CBOR
   encoded, the BLAKE3 genesis hash computed. The trustees jointly sign
   `BlsAggregate(GenesisState)`. The result is published on the
   `Ants-Community/ants` repository at a fixed path
   (`ants/spec/genesis-state/ants-v1-genesis.cbor`) plus its hash baked
   into the reference client binary.

3. **Bootstrap DHT seeds.** The trustees each run a peer instance with
   stable network addresses. The list of bootstrap addresses goes into
   the GenesisState `bootstrap_dht_seeds` field. Reasonable starting
   count: 3–5 globally-distributed addresses.

4. **Drand integration.** The chosen drand network (default: League of
   Entropy mainnet) is selected and its initial round value baked into
   the GenesisState. The launch timestamp is chosen to align with a
   known drand round to minimise initial degraded-seed activity.

5. **Embedding model packaging.** The reference client binary embeds the
   BGE-M3 weights and tokenizer files, hashed at build time, with the
   hash matching the GenesisState `weights_hash` and `tokenizer_hash`.

6. **Public launch.** The reference client binary is released. The first
   peers (trustees, then early-adopter community members) start their
   instances. The network is alive when the first non-trustee peer
   successfully joins, attests, and exchanges its first cache entry —
   the threshold being a single transaction with at least one trustee.

7. **The first 30 days.** Active monitoring of: peer-count growth,
   genesis-trustee influence ratio, drand outage frequency, fault-proof
   propagation latency (the `T_prop < T_beacon` constraint),
   bond-admission contention. Each is the target of a separate b2-style
   measurement specified in the relevant RFC.

8. **GOVERNANCE.md sunset checkpoint.** At 6 months, an honest
   assessment of progress against the v1.0 conditions (RFCs frozen for
   6 months, reference client audited, ≥1000 peers in 30 days, ≥3
   independent implementations, ≥15 active contributors). If any
   condition appears unreachable within 24 months, the BDFL publishes
   the candid assessment GOVERNANCE.md requires.

---

## Trustee key rotation

A trustee will need to rotate keys at least once during the bootstrap
window: hardware lost or upgraded, TEE attestation re-issued after a
vendor firmware update, suspected key compromise, or planned departure
with a successor designated. The GenesisState is signed once and never
amended in-protocol (per §Genesis state above), so naïvely a key change
would require a protocol-version bump and a coordinated migration —
operationally absurd as a response to a single hardware swap.

This section (added in v0.2) specifies an in-protocol key-rotation
mechanism that does **not** require a protocol-version bump. The
mechanism reuses primitives that already exist (L1 CRDT, BLS aggregate
signatures, TEE attestation) and adds no new ones.

### Trustee slot vs trustee key

The GenesisState `trustees[]` array fixes the **trustee slot**
identity: slot `i` is whoever the i-th array entry pointed to at
genesis. The slot is stable for the life of the protocol; the **key**
bound to that slot may rotate.

A peer's view of the current trustee set is computed by replaying all
finalised key-rotation announcements (defined below) against the
GenesisState. The result is a slot-indexed mapping
`slot_index → (current_peer_id, current_bls_pubkey, current_attestation)`
that every honest peer computes identically from the same inputs.

The granted-weight schedule `G(0)·e^{-δ_genesis · t}` is anchored to
**slot creation time** (genesis launch_timestamp), not to the current
key's age. Key rotation does not reset the sunset curve. A trustee
cannot earn fresh genesis weight by rotating; the slot decays on its
original schedule regardless.

### Key-rotation announcement

A trustee rotates keys by publishing a CBOR-encoded
`KeyRotationAnnouncement` to L1, structured as:

```
KeyRotationAnnouncement {
  slot_index:        u8,                       // index into GenesisState.trustees
  prev_peer_id:      Ed25519 pubkey,           // outgoing key (must match current slot binding)
  new_peer_id:       Ed25519 pubkey,
  new_bls_pubkey:    BLS12-381 G1 pubkey,      // derived per RFC-0008 §3.4
  new_attestation:   opaque vendor blob,       // fresh TEE attestation binding new_peer_id to hardware
  reason_code:       enum { HW_REPLACEMENT,
                            ATTESTATION_REFRESH,
                            SUSPECTED_COMPROMISE,
                            PLANNED_DEPARTURE,
                            OTHER },
  epoch:             u64,                      // L2 epoch at which announcement is filed
  nonce:             32-byte random,
  prev_sig:          Ed25519 signature over BLAKE3.derive_key(
                       "ants-v1-trustee-rotation", canonical-CBOR of preceding fields)
                     using the prev_peer_id key,
  witness_acks:      [ { acker_slot: u8, sig: Ed25519 }, ... ]
}
```

The new context string `"ants-v1-trustee-rotation"` is added to
RFC-0008 §4.1's reserved-context table.

The announcement is propagated through L1 alongside fault proofs (same
CRDT mechanism, different message type) and is **self-authenticating**
in the same sense: any peer can verify it independently against the
public state.

### Admission rules

An announcement is **admitted** (slot binding updated network-wide) when
**all** of the following hold:

1. **Authenticity.** `prev_sig` verifies under `prev_peer_id`, and
   `prev_peer_id` matches the current binding of `slot_index` from
   the network's replay of prior announcements (or the GenesisState
   entry if no prior rotation).
2. **Attestation.** `new_attestation` is a fresh, valid TEE
   attestation per RFC-0005, binding `new_peer_id` to attested
   hardware. Stale attestations are rejected.
3. **Witness threshold.** `witness_acks` contains valid Ed25519
   signatures from at least `⌈2/3 · (T − 1)⌉` of the **other** current
   trustees (T = total current trustees; the rotating slot does not
   count toward its own quorum). Each ack signs
   `BLAKE3.derive_key("ants-v1-trustee-rotation-ack",
   canonical-CBOR of the rotation announcement without witness_acks)`
   under the acker's current peer_id.
4. **Epoch freshness.** The L2 chain has reached `epoch` and not
   advanced beyond `epoch + ROTATION_ADMISSION_WINDOW` (default: 7
   epochs, calibratable). Announcements outside this window are
   rejected as stale or premature — the same anti-replay discipline
   bond admission uses.

   **Unit note (added in v0.3 Round 4 EDGE).** "Epoch" in this rule
   refers to the L2 `EPOCH_DURATION` constant (RFC-0008 §7, default
   24 hours), **not** the `POUH_BLOCK_TIME` (30 seconds). The 7-epoch
   default window is therefore approximately **1 week of wall-clock
   time**, not 3.5 minutes of block-clock time. Drand outages (RFC-0008
   §4.3) do not collapse this window: epochs continue to advance via
   the degraded-seed fallback, so the wall-clock duration of an
   admission window is robust to entropy-source outages of any duration
   the protocol survives. This unit clarification closes a misreading
   noted in the multi-persona Round 2 review (cold-eyed software
   engineer persona).

The 2/3-of-other-trustees threshold mirrors RFC-0004's L2 finality:
rotation is a coordination object across the trustee set, so the same
fault-tolerance bound applies.

### Compromise path

If `prev_peer_id` is compromised, the attacker can produce a valid
`prev_sig` over an announcement that rotates the slot to a key the
attacker controls. The witness-ack requirement is what prevents this:
the attacker also needs ⌈2/3 · (T − 1)⌉ honest trustees to
countersign, which they will not do without out-of-band confirmation.
The compromise reduces to the same honest-majority assumption RFC-0004
makes for L2 finality — no weaker, no stronger.

**Forced rotation without `prev_sig`** (the lost-key case): when a
trustee has lost access to `prev_peer_id` entirely, they cannot
self-rotate. Instead, the remaining trustees publish a
`TrusteeRevocation` (same envelope as KeyRotationAnnouncement but with
`prev_sig` omitted and `reason_code: SUSPECTED_COMPROMISE` or
`PLANNED_DEPARTURE`). Admission requires the **full** witness
threshold `⌈2/3 · (T − 1)⌉` plus an explicit `new_peer_id` field that
may equal `null` (slot left vacant) or a designated successor. A
revocation without successor reduces T by one; subsequent admission
thresholds recompute. This path is the in-protocol equivalent of a
governance action and is reserved for compromise or departure cases —
the witnesses must publish a brief signed justification alongside,
which lives in the L1 record.

### Equivocation

A trustee that signs **two different** key-rotation announcements at
the same `slot_index` and `epoch` is committing self-authenticating
equivocation (the two `prev_sig` values are the proof). The L1 fault
mechanism applies: the rotating slot is **frozen** — current binding
stays in force, no further rotation by `prev_peer_id` is admitted —
until a `TrusteeRevocation` resolves the slot to a clean successor or
vacancy. The slot's granted weight continues decaying on schedule
during the freeze.

#### Escape hatch from the freeze without synchronous coordination

*Added in v0.3 to address the deadlock case raised by the multi-persona
Round 2 review (applied cryptography persona): trustee compromise +
key loss + attacker equivocation locks the slot, and the standard
TrusteeRevocation path requires the remaining trustees to coordinate
out-of-band during a crisis — across jurisdictions that may not be
able to act rapidly.*

The standard `TrusteeRevocation` path (§Compromise path) remains
available. To remove the implicit "synchronous coordination" assumption,
**post-equivocation revocations may accumulate witness acks
asynchronously** over an extended window:

- The first remaining trustee to observe the equivocation publishes a
  `TrusteeRevocation` envelope (no `prev_sig`, reason_code
  `SUSPECTED_COMPROMISE`) with `witness_acks: []` initially. This
  envelope is **provisional**: it does not by itself revoke the slot.
- Other remaining trustees, each on their own clock and in their own
  jurisdiction, may append a witness ack to the provisional envelope
  by gossiping their signed `bond_admission_ack`-style addendum to L1
  at any point within the extended window.
- The window is **`EMERGENCY_REVOCATION_WINDOW`**, default
  **30 epochs (≈ 30 days at the nominal 24h epoch)**, calibratable per
  RFC-0008 §7. The window is longer than the standard
  `ROTATION_ADMISSION_WINDOW` because the worst case for which this
  exists is "the global trustees are in 5 jurisdictions, two are on
  holiday, one is in a country with poor connectivity for two weeks."
- The envelope becomes **final** the moment its accumulated acks reach
  `⌈2/3 · (T − 1)⌉`. Finality is observable to anyone replaying L1.
- If the window expires without enough acks, the provisional envelope
  expires; the slot remains frozen; any remaining trustee may
  re-initiate.
- Acks are themselves non-equivocable: a trustee that signs two
  conflicting `TrusteeRevocation` provisionals (e.g., one to vacate
  the slot, one to name a successor) commits its own equivocation
  fault and is itself frozen until cleaned up. The asynchronous
  accumulation does not weaken the anti-equivocation discipline.

The mechanism preserves the safety property of the standard path —
quorum of `⌈2/3 · (T − 1)⌉` of remaining trustees is still required to
remove a slot from freeze. What changes is the *coordination shape*:
acks may arrive asynchronously and from any jurisdiction without
real-time synchronisation. The honest framing: this does not weaken
the trust assumption, it widens the practical operating envelope of
the same assumption to match the global, multi-jurisdiction reality
of the trustee set.

### What this does not address

The mechanism handles individual trustee key changes. A coordinated
compromise of more than ⌈1/3 · T⌉ trustees defeats the witness
threshold — the same honest-majority assumption the rest of the
architecture rests on. That residual handoff is, again, the
credible-fork-threat.

The mechanism also assumes the L1 CRDT is functioning. During a
partition that isolates a rotating trustee, the announcement may
propagate on only one side; the §Partition recovery mechanism of
RFC-0004 v0.5 (Σ-T fork choice) reconciles after partition heal — the
fork with the witnessed announcement wins because the witnesses'
combined tenure is the larger Σ-T.

---

## What we have not figured out yet

- **Exact `δ_genesis` calibration.** The sunset curve's decay rate
  governs how quickly trustee influence drops below θ. Too fast and the
  network may collapse if non-trustee growth lags; too slow and trustees
  retain authority unnecessarily long. Default proposal: chosen so that
  `Φ_genesis(t = 1 year) ≈ θ` under expected growth, with the curve
  published in GenesisState. Empirical calibration via b2.
- **Bootstrap DHT seed liveness.** If all bootstrap seeds go offline
  (DDoS, ISP issues, jurisdiction problems), new peers cannot join. The
  protocol relies on existing peers to gossip up-to-date DHT seed lists,
  but no mechanism is specified for clients to update their embedded
  bootstrap list. Open.
- **Cold-start for late joiners.** A peer joining at `t = 1 year` must
  download `1 year × 30s` worth of L2 blocks = ~1M blocks. Block
  pruning policy is mentioned in RFC-0004 but not specified. State
  snapshots are the standard solution; needs spec.
- **Off-network bootstrap.** A peer behind a strict firewall, or in a
  jurisdiction with hostile network policy, may not be able to reach
  drand or the standard bootstrap seeds. Mitigation: rely on alternative
  network paths (Tor, I2P) — but the protocol does not currently specify
  these and they have their own dependencies.
- **`ROTATION_ADMISSION_WINDOW` calibration.** The default 7-epoch
  window (≈ 1 week given EPOCH_DURATION = 24h per RFC-0008 §7) trades
  off witness-ack collection time vs replay resistance. Tightening it
  stresses the coordination path; loosening it widens the equivocation
  window. Calibratable, b2-class.

---

## How to contribute to this document

Open an issue on `Ants-Community/ants` with the tag `rfc-0010`. Three
kinds of contribution most valuable:

- **Empirical phase-transition data**: once a testnet runs, the actual
  measured growth rates and capability-availability points feed back
  into the phase definitions here.
- **Failure-mode reports**: if the bootstrap sequence fails on a specific
  hardware or network configuration, file an issue with reproduction
  steps. This RFC will absorb the discovery.
- **Specific empirical proposals** for the open questions above:
  recommended `δ_genesis`, recommended block-pruning policy, recommended
  off-network bootstrap paths.

The document is CC0. Adopt this bootstrap pattern for your own
decentralised protocol if it helps.

---

*Drafted by the ANTS founding contributors, May 2026.
The first peer is the hardest. The hundredth is the test. The thousandth is the success.*
