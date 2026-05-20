# RFC-0004 — Reputation Chain with Proof-of-Unique-Hardware Consensus

**Status:** Draft · v0.1 (early — substantial revision expected)
**Topic:** A small, slow, immutable blockchain for recording peer misbehaviour. A new consensus mechanism that is neither Proof-of-Work nor Proof-of-Stake.
**Audience:** You, if you have read enough blockchain whitepapers to be skeptical of one more.
**Depends on:** [RFC-0003](./RFC-0003-verification.md), [RFC-0005](./RFC-0005-identity.md)

---

## What this document specifies

The community-layer ledger (RFC-0001) lives inside each peer pair. It does not need global consensus, because the ledger is local. The cache (RFC-0002) lives across many peers as a distributed key-value store. It does not need global consensus, because peers can hold slightly different views and resolve queries individually.

But *some* facts about peer behaviour must be global, immutable, and adversarially trustworthy. When peer X is caught fabricating outputs three times in a week, the network must agree that this happened — across all peers, durably, even if the original verifiers are themselves later removed. That requires a blockchain.

This document specifies a small, slow, deliberately unambitious blockchain for *exactly that purpose* — and proposes a new consensus mechanism specifically suited to it.

---

## What goes on chain — and what does not

The reputation chain records, exclusively, **misbehaviour facts**. Specifically:

- Verified instances of fabricated output (from Tier 3 verification failures)
- Repeated patterns of cache poisoning
- Identity violations (attempted Sybil attacks, attested-hardware reuse)
- Slashing events (the protocol's response to the above)
- Validator misconduct

The chain does *not* record:

- Routine peer-to-peer transactions (those live in local ledgers)
- Cache contents (those live in the DHT)
- Service histories or reputation scores (those are computed locally from chain events plus local observation)
- Anything resembling a financial transaction (the chain has no notion of payments)

The chain is a *log of bad facts*, not a record of all activity. Its volume is small — perhaps a few thousand events per day at network maturity, not millions per second. This shapes everything about its consensus design: we do not need high throughput, we need high integrity.

---

## Proof-of-Unique-Hardware consensus

The chain uses a consensus mechanism we are calling **Proof-of-Unique-Hardware** (PoUH). It is genuinely new — neither Proof-of-Work, nor Proof-of-Stake, nor any of the BFT variants we have surveyed. We claim it as an original contribution of ANTS.

### The core idea

Every peer in the network proves, via [hardware attestation](./RFC-0005-identity.md), that it is running on a unique physical CPU. The protocol counts each attested unique CPU as exactly one voice in consensus — no more, no less. Validators are selected pseudorandomly from the set of all attested peers, weighted equally per CPU.

This sidesteps the three problems that haunt other consensus mechanisms:

- **Not Proof-of-Work**: no energy waste. Validators run light cryptographic operations, not hash competitions.
- **Not Proof-of-Stake**: no plutocracy. The richest peer is exactly as influential in consensus as the poorest, as long as both have attested hardware.
- **Not delegated BFT**: no permissioned validator set. The validator pool is the entire attested peer population.

### Validator selection

For each new block, a deterministic pseudorandom function selects a committee of K attested peers (default K=64) from the global attested population. Selection uses a Verifiable Random Function (VRF) seeded by the previous block's hash plus the block height.

Properties:
- **Deterministic**: any peer can verify, given the seed, that the correct committee was selected.
- **Unpredictable in advance**: no peer can predict the next committee until the previous block is finalised — preventing pre-coordination attacks.
- **Uniform**: every attested peer has an equal probability of being selected for any given block, summed across all blocks proportional to 1/(network size).

A committee of 64 from a network of (say) 10,000 attested peers means each peer is selected for committee duty roughly once every 156 blocks. At a 30-second block time, that is once every ~78 minutes. Burden is real but small.

### Block production

Within the committee for a given block:
1. One member is designated proposer (deterministically, by VRF).
2. The proposer collects pending misbehaviour-facts from the mempool, validates each one (against the verification mechanisms in RFC-0003), and assembles them into a candidate block.
3. The candidate block is broadcast to the rest of the committee.
4. Each committee member verifies the block's contents and signs an attestation.
5. When 2/3 (rounded up) of committee signatures are collected, the block is finalised.
6. The block is broadcast to the entire network, where any peer can verify the committee signatures.

Block time target: 30 seconds. This is slow by modern blockchain standards. We choose slow on purpose — there is no urgency, and slow makes adversarial conditions less profitable.

### Finality

A block is finalised when 2/3 of its committee signs. There is no probabilistic finality (unlike Bitcoin) and no challenge period (unlike optimistic rollups). The chain is monotonic: once a block is in, it stays.

A block that fails to reach 2/3 within a timeout (default 120 seconds) is skipped; the next block's committee proceeds with the same pending misbehaviour-facts.

---

## What makes PoUH work

The security of PoUH rests on three pillars:

**1. Hardware attestation prevents Sybil.** A peer cannot pretend to be many peers without acquiring many physical CPUs with attestation chains. The cost of fake identities is bounded below by hardware cost. From our earlier analysis: to control 10,000 attested identities via cloud confidential compute, an attacker spends $7-12M per year on hardware rentals.

**2. Validators have skin in the game without staking.** A validator who signs a fraudulent block is caught when honest validators dispute it, and is slashed — their attestation is revoked from the chain, their NCS earnings are zeroed, and their hardware identity is permanently flagged. They lose their ability to participate. No staking deposit was required, but the consequence of bad behaviour is loss of *future income from being a peer*. This is sufficient incentive against attack as long as honest participation is economically valuable.

**3. Committee rotation makes long-term corruption impossible.** Even if an attacker corrupts an entire committee for one block, the next block's committee is a new random sample. Sustained corruption requires sustained 67%+ control of the global attested peer population — which costs more than the gain from any plausible attack.

### Attack costs

A simplified threat model:

- **To censor one block** (prevent a misbehaviour-fact from being recorded): the attacker needs >1/3 of the specific committee. At K=64, that means 22 committee members must be the attacker's. The probability that a randomly-selected committee of 64 from a 10,000-peer network has 22 attacker-controlled members, when the attacker has corrupted 10% of all peers, is computable from the hypergeometric distribution; it's vanishingly small. The attacker would need ~30% of the entire network to have a reasonable chance.
- **To corrupt the chain history** (rewrite finalised blocks): impossible. Once 2/3 of a committee signs, the block is final.
- **To pause the chain** (force timeouts): an attacker with >1/3 of the network can prevent any block from achieving 2/3. The chain stops but is not corrupted. Honest peers reconvene under a recovery protocol (specifications pending).

The attack cost calculation gives roughly: at 30% of network corruption ($21-36M/year in our threat model), the attacker can pause the chain. At 67%+, they can corrupt it. This is *much higher* than the value of any single misbehaviour event the chain protects. The economics defeat the attack before it begins.

---

## Slashing

When a misbehaviour-fact is finalised on chain, consequences follow according to severity:

| Severity | Trigger | Slashing consequence |
|---|---|---|
| **Soft** | 1-2 verified fabrication events in 30 days | Quality score downweighted by 0.1; eligible for full recovery after 90 clean days |
| **Medium** | 3-5 verified events in 30 days, or 1 cache-poisoning event | Quality multiplier × 0.5; cache write permission suspended for 30 days |
| **Hard** | 6+ events in 30 days, or repeated cache poisoning, or validator misconduct | Cache write permanently revoked; NCS earning capped at 25% of peer norm for 180 days; attestation flagged for review |
| **Terminal** | Confirmed Sybil identity, deliberate consensus attack, or repeated hard slashing | Hardware identity permanently flagged. Cannot rejoin under the same attestation. New hardware required to re-enter the network. |

Slashing is not reversible by any single authority. A slashed peer may challenge the slashing event by initiating a dispute, which goes through a separate dispute-resolution committee (specifications pending).

---

## Why we did not pick something else

A short tour of paths we considered and rejected, for transparency.

**Pure DAG (no consensus)** — we considered this initially. A DAG of signed reputation events, no global ordering, peers compute their own views. Attractive because it's cheaper. Rejected because reputation events that are not globally agreed-upon are gameable: a peer can pretend the slashing event happened in a different order, or that conflicting events nullify each other, and there is no shared truth to appeal to. Reputation *needs* some globally agreed-upon facts.

**Standard BFT with fixed validator set** — fast and well-understood, but requires designating a fixed validator set, which is centralisation by another name. Rejected.

**PoW** — wasteful. Solves a problem we don't have.

**PoS** — plutocratic. Solves a problem we don't have, and creates one we explicitly refuse (token capture).

**Avalanche / Snowball** — interesting but novel, complex, and untested at adversarial scale outside the AVAX ecosystem. We are willing to be novel only where novelty is necessary. PoUH is the novelty we accept; we don't want to compound it.

**Delegated PoS variants (Cosmos, Tendermint families)** — still plutocratic at the delegation level. Rejected.

PoUH is what's left when you require global consensus, no token, no energy waste, no plutocracy, no permissioned validator set, and Sybil resistance.

---

## Privacy on the chain

Misbehaviour-facts are public by design. The chain records that *peer X with attested identity Y was caught fabricating output Z*. Anyone can read it.

Some privacy mitigations:

- The original prompt that led to a misbehaviour-fact is *not* recorded on chain — only its embedding, the response hash, and the verifier's attestation of the disagreement. The full prompt is not retrievable from chain inspection alone.
- Hardware-identity public keys on chain are not linkable to real-world identity (no KYC layer). They are linkable to the same peer's *future behaviour*, which is exactly the point.
- Slashing events do not name human operators. They name attested hardware identities.

Operators who do not want their slashing events visible should not get slashed. The chain is intentionally a public shaming mechanism.

---

## What we have not figured out yet

Significant open problems:

- **Committee size.** K=64 is a guess. Smaller is faster but less robust; larger is the reverse.
- **Block time.** 30 seconds is conservative. May be too slow if misbehaviour-fact volume grows; may be too fast if network conditions degrade.
- **Recovery from network partition.** If the network splits, do we have one chain or two? Standard answer: longest-valid-chain wins, but this needs careful spec.
- **Dispute resolution mechanism.** A slashed peer challenging their slashing — how does this work? Probably a separate committee with higher K and longer review period, but it's undefined.
- **State growth.** Even at modest event volume, the chain grows. State pruning policy (archive nodes vs full nodes) is undefined.
- **Bootstrap committee.** When the network is new and has few attested peers, the committee can't be 64. What's the bootstrap policy? Probably: protocol launches with the founders' attested peers serving as bootstrap committee, transitioning to random selection as the attested population grows past 200.
- **Verification of validator hardware over time.** Hardware attestations can be revoked by vendors. How does the chain handle a validator whose hardware attestation expires mid-tenure? Specifications pending.
- **PoUH vulnerability to vendor compromise.** If a hardware vendor (Intel, AMD) is compromised at the attestation-signing layer, an attacker could mint unlimited fake identities. Mitigation: multi-vendor attestation, with chain weighting that downweights any single-vendor majority. This is critical to RFC-0005 and to the chain's long-term security.

---

## Adversarial considerations

**Committee bias attack.** An attacker tries to influence VRF seeding to land their peers in many committees. Mitigation: the VRF seed combines previous block hash, block height, and a beacon value derived from external entropy sources (publicly verifiable randomness — e.g., the drand beacon network). Manipulating the seed requires manipulating *external* entropy, which is much harder.

**Cabal of validators.** A coordinated subset of validators tries to censor specific peers' misbehaviour-facts from being recorded. Mitigation: if a misbehaviour-fact is submitted but consistently fails to appear in blocks, after N committee rotations the chain logs a censorship event, and the next several committees prioritise the suppressed fact.

**Spam attacks.** An attacker submits massive volumes of bogus misbehaviour-facts to choke the chain. Mitigation: submitting a misbehaviour-fact requires paying a small NCS fee (paid to the committee that validates it). The fee is small enough not to discourage legitimate reports but large enough to make spam economically prohibitive.

**Hardware vendor compromise.** Discussed above. The most serious threat in the medium term; mitigated by multi-vendor attestation in RFC-0005 and by long-term investment in open-hardware TEEs.

---

## How to contribute to this document

Open an issue on `Ants-Community/ants` with the tag `rfc-0004`. Particularly valuable contributions: formal analysis of PoUH's security properties, comparisons against more obscure consensus mechanisms we may not have considered, empirical data from existing distributed systems on committee-size optimality.

The document is CC0.

---

*Drafted by the ANTS founding contributors, May 2026.
A blockchain that records only what must be recorded.
A consensus that values every honest CPU equally.*
