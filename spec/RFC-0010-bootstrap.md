# RFC-0010 — Bootstrap Sequence

**Status:** Draft · v0.1
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
- **Trustee key rotation during bootstrap.** A trustee may need to
  rotate keys (compromise, departure). Currently undefined; the
  GenesisState is supposed to be immutable, so trustee key rotation
  effectively requires a protocol-version bump. Whether to allow
  in-protocol trustee key rotation via L1 attributable-fault-style
  signed announcements is open.
- **Cold-start for late joiners.** A peer joining at `t = 1 year` must
  download `1 year × 30s` worth of L2 blocks = ~1M blocks. Block
  pruning policy is mentioned in RFC-0004 but not specified. State
  snapshots are the standard solution; needs spec.
- **Off-network bootstrap.** A peer behind a strict firewall, or in a
  jurisdiction with hostile network policy, may not be able to reach
  drand or the standard bootstrap seeds. Mitigation: rely on alternative
  network paths (Tor, I2P) — but the protocol does not currently specify
  these and they have their own dependencies.

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
