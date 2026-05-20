# RFC-0005 — Identity

**Status:** Draft · v0.1 (early — substantial revision expected)
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

Attestation reports have **expiration windows** (typically a few weeks to a few months, depending on vendor). When a report expires, the peer must obtain a fresh one. This handles two problems: (a) a stolen TEE that has been remotely repaired or replaced gets a new attestation; (b) a compromised vendor key can be retired by waiting for old attestations to expire.

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

---

## The long-term open-hardware path

A protocol that depends permanently on Intel, AMD, ARM, and Apple is dependent on four specific corporations. This is acceptable in 2026, where these vendors have the only mature TEEs. It is not acceptable as a permanent state.

The protocol's identity layer commits to migrating, over the next decade, toward open-hardware TEE roots. Specifically:

- **RISC-V Keystone** is a research effort to build a fully open-hardware TEE on RISC-V cores. Not production-ready in 2026, but advancing.
- **OpenTitan** (Google + lowRISC + Western Digital + others) is an open-source silicon root-of-trust. Some commercial implementations exist; broader adoption pending.
- **MIT's open secure enclave research** and several academic projects are exploring formally-verified TEE architectures.

The protocol commits to:

1. Accept open-hardware TEE attestations *as soon as* they are production-grade. Currently planned: 2027–2028 for Keystone, OpenTitan acceptance.
2. Weight open-hardware attestations *higher* than closed-vendor attestations in reputation calculation, as a deliberate incentive for migration.
3. By v2.0 (long after v1.0), the protocol intends to require *at least one* open-hardware attestation for validator participation — eliminating the dependence on closed corporate roots-of-trust at the most security-critical layer.

This is a multi-year migration. It is included in this RFC because the alternative — pretending closed hardware is permanently acceptable — would be dishonest.

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

**TEE vulnerability discovery.** A new side-channel attack (similar to historical examples like Spectre/Meltdown) allows extracting attestation keys from a TEE. Mitigations: most attacks require physical or co-located access, limiting scale. Vendors release firmware patches; old vulnerable peers are flagged.

**Hardware re-attestation farming.** An attacker buys 1000 TEE-capable devices once and re-uses them indefinitely. Defence: this is not really an attack — the attacker has 1000 identities for the cost of 1000 devices, which is the expected pricing. Nothing to mitigate.

**Stolen TEE.** A laptop is stolen, the thief tries to use its ANTS identity. Defence: the identity is bound to the TEE, which the thief now has. But the original owner can publish a revocation event (signed by their own backup key, which the protocol provides at identity creation time). The stolen identity is then untrusted.

---

## How to contribute to this document

Open an issue on `Ants-Community/ants` with the tag `rfc-0005`. Particularly valuable: cryptographic analysis of the multi-vendor weighting scheme, empirical data on cloud TEE costs, alternative Sybil-resistance mechanisms we have not considered.

The document is CC0.

---

*Drafted by the ANTS founding contributors, May 2026.
One CPU, one voice. The hardware is the identity.*
