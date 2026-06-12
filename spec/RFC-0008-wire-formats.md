# RFC-0008 — Wire Formats, Crypto Primitives, and Reference Constants

**Status:** Reference · v0.7 (living standard)
**Topic:** The concrete byte-level choices the protocol makes once and refers to from everywhere else.
**Audience:** You, if you are about to write code.
**Depends on:** all design RFCs (this document is the implementation glue between them).

---

## What this document is, and what it is not

The design RFCs (0001 through 0006) specify *what* the protocol does and *why*.
They do not specify *which bytes go on the wire*. Without that, two
well-intentioned implementations of the same RFC will not interoperate.

This document fixes the engineering choices that every RFC silently depends on:
serialization, hash functions, signature schemes, the pseudorandom-function and
verifiable-random-function used for beacon-derived selection, the encoding of
NCS as integers, and the pinning of the canonical embedding model. It also
collects the parameter constants that today live scattered across the design
RFCs, with explicit markers for which are *pinned* (must be the same across all
peers) and which are *calibratable* (will be revised after testnet evidence).

This document is **prescriptive**, not exploratory. It changes when a primitive
must change — and every such change is a coordinated breaking event. It does
not get "argued with" in the same sense the design RFCs do; what it gets is
*replaced cleanly* when a better primitive is needed.

Test vectors live in a sibling repository (`ants-test-vectors`, to be created
when the reference client begins). This document specifies their shape; the
vectors themselves are generated against the reference implementation and
versioned alongside it.

---

## Why we pin

Three things become possible once primitives are pinned and only when they are
pinned:

- **Interoperability across independent implementations.** The v1.0 condition
  for at least three independent client implementations passing a conformance
  test suite requires every implementation to produce byte-identical outputs
  for the same input. Without pinning, conformance is undefined.
- **Auditable cryptographic claims.** Statements like "Type-I error ≤ α" or
  "the beacon is unpredictable" or "fault proofs are self-authenticating"
  are claims about *specific* mathematical objects. Until the objects are
  named, the claims are vibes.
- **A test-vector pack.** With pinned primitives, every protocol step has a
  fixed-input/fixed-output specification an implementer can verify against
  before deploying anything. This is the cheapest possible way to ensure
  early implementations agree.

---

## 1 · Serialization

### 1.1 · Protocol-level objects: deterministic CBOR

**All protocol-level objects** — commit-at-send objects, fault proofs, attestation
wrappers, cache entries, beacon-derived selection inputs, anything that is
hashed or signed — are encoded as **CBOR per [RFC 8949](https://www.rfc-editor.org/rfc/rfc8949)
with deterministic encoding per §4.2.1**:

- shortest-form integer encoding;
- definite-length encoding for arrays, maps, and byte strings;
- map keys sorted by the bytewise lexicographic order of their canonical CBOR
  encoding;
- no indefinite-length items;
- no tags except those explicitly listed in this RFC (currently: tag 0
  date-time strings, tag 32 URI, tag 42 IPLD CID — only where needed).

The motivation is brutally pragmatic: CBOR is binary (compact on the wire),
schema-less (forward-compatible with payload evolution), and has a documented
deterministic encoding (essential because we hash and sign these objects).
Hand-rolled codecs are 200–500 lines per language, which is acceptable for the
implementation diversity the v1.0 goal requires.

The concrete message-level encodings built on this profile — the DHT RPCs,
gossip frames, fault proofs, chain objects, commits, receipts, and bond
admissions the reference client exchanges — are catalogued in §11
(added in v0.6).

### 1.2 · Human-facing and developer-facing: JSON

**Anything intended for human eyes or developer tooling** — client API
responses, CLI output, log lines, error messages, the public read-only view of
the reputation chain — uses **standard JSON (RFC 8259)**.

JSON is never hashed or signed. If a JSON object needs to be hashed, it is
first canonicalised to deterministic CBOR per §1.1.

### 1.3 · Numeric types

- Integers: CBOR major types 0 (unsigned) and 1 (negative). Standard sizes.
- Booleans: CBOR `true` / `false` (major type 7, values 21/20).
- Byte strings: CBOR major type 2. No string encodings for binary data.
- Text strings: CBOR major type 3, UTF-8.
- Floating point: **forbidden in protocol-level objects** (use fixed-point
  integers; see §6 NCS encoding for the canonical pattern). Allowed only in
  JSON for human-facing values where precision is not security-relevant.

---

## 2 · Hash functions

### 2.1 · Protocol-internal: BLAKE3

All protocol-internal hashing — Merkle trees over logit traces (RFC-0003),
content-addressing of cache entries (RFC-0002), fault-proof binding (RFC-0004),
peer-identity content hashes — uses **[BLAKE3](https://github.com/BLAKE3-team/BLAKE3-specs/blob/master/blake3.pdf)**
with **32-byte output** (256-bit security) in **default mode**.

BLAKE3 is chosen over SHA-256 because:
- It is 5–10× faster on commodity hardware (relevant for Merkle hashing over
  long logit traces, hundreds of leaves per commit-at-send).
- It is parallelisable by design (matters for cache shard re-verification on
  multi-core CPUs).
- Its `derive_key` mode gives us a clean PRF primitive (see §4 below).

### 2.2 · TEE-attestation-chain compatibility: SHA-256 / SHA-384

The TEE attestation chains produced by Intel TDX, AMD SEV-SNP, ARM CCA, Apple
Secure Enclave, and Qualcomm QSEE use **SHA-256** or **SHA-384** by vendor
mandate. The protocol accepts these signatures *as they are*; no re-hashing is
performed for compatibility purposes.

Where a protocol-level object must include an attestation, the protocol-level
object's outer wrapper uses BLAKE3 (per §2.1), and the attestation is included
verbatim as an opaque byte string inside it. The two hash universes do not mix.

The same compatibility rule covers the **drand** beacon (§4.2–§4.3, added in
v0.6): drand's chained-round relations are SHA-256 by drand's own protocol —
the round message is `SHA-256(previous_signature ‖ round_be64)` and the
published randomness is `SHA-256(signature)` — and the protocol verifies them
as such (§11.7). Protocol-internal derivations over the *verified* randomness
(the VRF seed, §4.2) use BLAKE3 per §2.1.

### 2.3 · Forbidden

- SHA-1, MD5: cryptographically broken, never used.
- Truncated hashes for security-relevant purposes: never.

---

## 3 · Signature schemes

### 3.1 · Peer identity signatures: Ed25519

All peer-to-peer signatures — signed cache entries, fault proofs, attestation
wrappers, beacon-published values, audit verdicts — use **Ed25519** per
[RFC 8032](https://www.rfc-editor.org/rfc/rfc8032).

Ed25519 is chosen because: it is deterministic (no nonce reuse risk), fast,
small (64-byte signatures, 32-byte keys), simple to implement correctly, and
nearly universally supported.

Tier 3 verification committees (RFC-0003) use Ed25519 individual signatures —
not BLS aggregation. Tier 3 committee size is small (N=3–7), so aggregation
brings no meaningful saving and Ed25519's lower per-verify cost dominates.

### 3.2 · TEE attestation chains: ECDSA-P256 (vendor-determined)

TEE vendors sign attestation quotes with **ECDSA over NIST P-256** in most
cases, occasionally RSA-2048 or RSA-3072. The protocol accepts these as the
vendor specifies them. The verification path is defined per vendor in RFC-0005
v0.2 (forthcoming).

### 3.3 · L2 PoUH committee signatures: BLS12-381

The L2 PoUH chain (RFC-0004 v0.3) finalises blocks when 2/3 of a K=64-member
committee signs. Naive ECDSA gives 64 separate signatures per block — too large
to gossip efficiently and slow to verify. Instead, the protocol uses
**BLS12-381 signatures with the `BLS_SIG_BLS12381G2_XMD:SHA-256_SSWU_RO_NUL_`
ciphersuite** per [IETF draft-irtf-cfrg-bls-signature](https://datatracker.ietf.org/doc/draft-irtf-cfrg-bls-signature/).

BLS signatures aggregate: 64 signatures over the same block hash become *one*
signature, verifiable in roughly the time of two ordinary ECDSA verifications.
The chain header carries the aggregate signature plus a bitmap of which
committee members signed.

**Bootstrap K transition (added in v0.2).** At bootstrap, committee size K is
small (typically 5–16). At small K the aggregation benefit of BLS over plain
Ed25519 multi-signature is negligible: 5 Ed25519 signatures occupy 320 bytes
(comparable to one BLS signature plus its bitmap), and BLS per-verify is
~10ms vs ~0.05ms for Ed25519. The protocol therefore transitions signature
scheme based on K: at **K ≤ 16** committee members sign the block hash
individually with Ed25519 (the block header carries K Ed25519 signatures plus
the member-ID list); at **K > 16** BLS12-381 aggregate signatures apply as
specified above. The K=16 threshold is calibrated so that BLS aggregation
provides ≥3× size and verification-cost reduction over Ed25519 multi-sig at
the crossover. It is **Calibratable** per §7.

**Disambiguation of "K" (added in v0.4).** *Closes a concern raised by
the multi-persona Round 2 review (applied-cryptography persona): the
spec did not disambiguate whether "K" in the transition rule refers to
the selected committee size or to the effectively-signing count.*

**`K` is the VRF-selected committee size, not the effectively-signing
count.** Specifically:

- The L2 protocol's epoch start (RFC-0004 §Layer 2) selects K committee
  members from the attested peer population via ECVRF (RFC-0008 §4.2,
  ELL2 since v0.3). `K = POUH_COMMITTEE_SIZE_K` per §7, scaling per
  RFC-0010 phase but deterministic from the epoch seed and the attested
  population. `K` is fixed at epoch start and known to every peer.
- The signature scheme for blocks within that epoch is determined by
  `K`: `K ≤ BLS_TRANSITION_K` → Ed25519 multi-sig; `K > BLS_TRANSITION_K`
  → BLS aggregate. The choice does not change mid-epoch.
- 2/3 finality is computed against `K`, not against the
  effectively-signing count. If `K = 17` and 4 members are offline,
  finalisation requires `⌈2·17/3⌉ = 12` signatures from the remaining
  13; this finalises the block under the BLS scheme since `K = 17 >
  BLS_TRANSITION_K = 16`.
- **Atomicity at the boundary.** An epoch with `K = 16` finalises every
  block of that epoch under Ed25519 multi-sig. The next epoch may have
  `K = 17` per the phase schedule (Phase 3 → Phase 4 transition or
  scale-out), in which case every block of that epoch uses BLS.
  There is no in-epoch hybrid; each epoch is uniformly one scheme.
- **The proposer does not choose.** Scheme selection is deterministic
  from `K`; the proposer reads the epoch's `K` from the epoch summary
  and produces a block header in the corresponding format. A proposer
  that produces an out-of-scheme header is committing an attributable
  fault (block rejected by all honest peers, equivocation-style
  evidence).

The honest framing: the v0.2 spec was correct but underspecified. The
disambiguation above makes the boundary atomic at epoch granularity and
removes the proposer's degree of freedom — what looked like ambiguity in
the original text was a missing sentence about epoch-granularity
atomicity, not a missing mechanism.

### 3.4 · One key per role

Each peer maintains, inside its TEE:
- One **Ed25519 keypair** as its long-term peer identity (used for §3.1).
- One **BLS12-381 keypair**, derived deterministically from the Ed25519 key,
  used exclusively when serving as a PoUH committee member (§3.3).

The derivation: `BLS_priv = BLAKE3.derive_key(context="ants-v1-bls-derive",
key_material=Ed25519_priv).first_32_bytes`.

The vendor's TEE-attested signing key (ECDSA-P256 or similar) is *outside* the
protocol's key hierarchy: it signs the attestation that binds the Ed25519
public key to the hardware, and that is all.

---

## 4 · PRF and VRF

### 4.1 · PRF for deterministic derivation: BLAKE3.derive_key

Where the protocol needs a pseudorandom function over arbitrary inputs —
deriving beacon-bound challenge positions, deriving auditor selection, deriving
sub-keys, computing canonical content addresses — it uses
**`BLAKE3.derive_key(context, key_material) → 32 bytes`** in its keyed-hash
mode.

The `context` parameter is a domain-separation tag (a constant ASCII string).
This document reserves the following context strings:

| Context | Used for | Defined in |
|---|---|---|
| `"ants-v1-beacon-audit-flag"` | deciding whether a commit gets audited | RFC-0003 §Tier 1 |
| `"ants-v1-beacon-position-set"` | deriving the challenge position set S | RFC-0003 §Tier 2 |
| `"ants-v1-beacon-auditor"` | selecting the auditor from the verifier set | RFC-0003 §Tier 2 |
| `"ants-v1-shard-key"` | computing the LSH shard key from an embedding | RFC-0002 §DHT routing |
| `"ants-v1-bls-derive"` | deriving the BLS key from Ed25519 | §3.4 above |
| `"ants-v1-fault-merkle"` | Merkle leaf hashing for fault proofs | RFC-0004 §Layer 1 |
| `"ants-v1-logit-trace-leaf"` | per-position logit-distribution leaf hashing | RFC-0003 §commit-at-send |
| `"ants-v1-logit-trace-root"` | root commitment over the full logit trace | RFC-0003 §commit-at-send |
| `"ants-v1-cache-entry-hash"` | content addressing of `Y_canon` in cache entries | RFC-0002 §"What the response field contains" |
| `"ants-v1-vrf-seed-degraded"` | fallback VRF seed when drand is unavailable | §4.2 below |
| `"ants-v1-trustee-rotation"` | binding for trustee key-rotation announcement | RFC-0010 §Trustee key rotation |
| `"ants-v1-trustee-rotation-ack"` | binding for trustee key-rotation witness ack | RFC-0010 §Trustee key rotation |
| `"ants-v1-vrf-seed"` | the per-block VRF seed (live drand path) | §4.2 below |
| `"ants-v1-genesis"` | the genesis-state hash | RFC-0010 §Genesis state |
| `"ants-v1-bond-tiebreak"` | race-safe bond-admission tie-break key | RFC-0004 §Atomicity; §11.10 |
| `"ants-v1-receipt-bag-root"` | binding a receipt-bag Merkle root to its peer | RFC-0004 §Selective disclosure; §11.9 |

New contexts are added to this table via amendment to this RFC, never
hard-coded ad-hoc. The domain-separation discipline is non-negotiable: re-using
a derived key across contexts is the single most common cause of subtle
cryptographic bugs in deployed systems.

**Binding status (added in v0.6).** The reference implementation binds
`"ants-v1-vrf-seed"`, `"ants-v1-vrf-seed-degraded"`, `"ants-v1-bond-tiebreak"`,
and `"ants-v1-receipt-bag-root"` today. Two groups of reserved strings are
*not* bound by the reference encodings: the beacon-challenge contexts
(`-beacon-audit-flag`, `-beacon-position-set`, `-beacon-auditor`) are
superseded by the keystream construction specified in §11.8, and the Merkle
contexts (`-fault-merkle`, `-logit-trace-leaf`, `-logit-trace-root`) by the
prefix-byte domain separation specified in §11.4 — in both cases the
implemented construction was pinned first and is what an independent
implementation must match; the strings remain reserved, never recycled. The
remaining contexts (`-shard-key`, `-cache-entry-hash`, `-bls-derive`, the
trustee-rotation pair, `-genesis`) await their implementing components.
Whether to migrate the superseded constructions onto `derive_key` for
uniformity is an open question (§10): it would re-pin every committed root
and challenge, so it happens before first cross-implementation interop or
not at all.

### 4.2 · VRF for committee selection: ECVRF-EDWARDS25519-SHA512-ELL2

Committee selection for the L2 PoUH chain (RFC-0004 §Layer 2) uses a
**Verifiable Random Function** per [IETF RFC 9381](https://www.rfc-editor.org/rfc/rfc9381),
specifically the ciphersuite **ECVRF-EDWARDS25519-SHA512-ELL2** (Elligator 2
hash-to-curve).

The chain header's previous-block-hash plus block height plus an external
entropy beacon contribution feeds the VRF input. The VRF output is verifiable
by anyone with the proposer's public key: anyone can confirm that the correct
committee was selected from the attested peer population, deterministically.

**Why ELL2 and not TAI (added in v0.3).** RFC 9381 documents two
hash-to-curve constructions for the EDWARDS25519-SHA512 family:
**TAI** (try-and-increment) iterates with a counter until a hash falls on
the curve — the iteration count depends on the input, so the wall-clock
time of `ECVRF_prove` and `ECVRF_verify` is input-dependent and leaks
information through cache and branch-prediction side-channels. **ELL2**
(Elligator 2) maps any field element to a curve point in a single
constant-time operation. The output distribution and verifiability are
identical; the only difference is timing. Because the protocol's VRF
inputs (previous-block-hash, height, drand round) are public, the
practical leak is bounded, but pinning a primitive whose constant-time
variant is freely available and equally interoperable would have been
gratuitous. We pin ELL2.

External entropy: the protocol uses **drand** (League of Entropy beacon) as its
public unbiasable randomness source for VRF seed contribution. Each block's
VRF seed is `BLAKE3.derive_key("ants-v1-vrf-seed", previous_block_hash ‖
block_height_be64 ‖ drand_round_value)`, where **`drand_round_value` is the
round's 32-byte published `randomness`** — `SHA-256` of the round's BLS
signature, the round's canonical public output (pinned in v0.6; the signature
itself is what gets verified, §11.7). drand's threshold-BLS construction
makes the value unforgeable by any party with less than 2/3 of the drand
network — substantially stronger than what ANTS can do on its own at bootstrap.

### 4.3 · Drand failover (added in v0.2)

drand outages have happened and will happen. Rather than stall the chain
(RFC-0004's L2 cannot finalise blocks if proposers cannot compute a VRF
seed), the protocol specifies a degraded-seed fallback:

- The proposer of block *N* attempts to fetch the drand round whose
  published time is closest to but not after the block's proposal
  timestamp.
- If no drand round value is available within a **30-second timeout**,
  the proposer constructs the VRF seed from
  `BLAKE3.derive_key("ants-v1-vrf-seed-degraded", previous_block_hash ‖
  block_height_be64)` alone, and sets `degraded_seed: true` in the block
  header.
- A block with `degraded_seed: true` finalises normally (committee signs
  per §3.3) but is *visibly flagged* in the chain record. Any peer
  reading the chain can see which blocks used degraded entropy.
- The 30-second timeout is a **hard floor**: a proposer that submits a
  degraded-seed block without honestly waiting that long is committing
  an attributable fault.
- Recovery is automatic: as soon as drand resumes, the next block's
  proposer uses the live drand value normally. Degraded-seed blocks
  remain in the chain history as a record of the outage.

The security implication is real and bounded: during drand outages, the
next committee is predictable to an attacker who can predict
`BLAKE3(previous_block ‖ height)`. The previous-block-hash is
unpredictable to an attacker if at least one honest peer signed the
previous block — the standing assumption. An attacker with 1/3 of the
*previous* committee can grind toward favourable next-committees during
drand outages by manipulating their share of the previous block's
content. This is a partial security regression during outages, bounded
by their duration. The honest framing: drand outages are operationally
costly even though they do not stall the chain.

#### Quantifying the grinding compounding bound

*Added in v0.4 Round 4 EDGE to close the concern raised by the
multi-persona Round 2 review (applied-cryptography persona): the v0.3
spec said "security regression bounded by outage duration" but did not
quantify the bound, and the compounding behaviour across multiple
consecutive degraded blocks was not specified.*

The attacker's grinding power per degraded block is bounded by the
number of distinct block contents they can produce before the block
deadline, which is itself bounded by their controlled hash rate times
the block time. For 1/3 of a committee with ordinary commodity hardware,
realistic grinding power per block is roughly **`G ≈ 2²⁰` block-content
candidates per block** (one million attempts in a 30-second block
under modest compute). This is generous to the attacker — most
realistic adversaries will be well below this.

For each degraded block, the attacker's grinding allows them to bias
the next committee selection by approximately
**`Δ ≈ sqrt(G · ln 2 / K)` additional members** in the worst case (a
maximum-search bound over `G` independent VRF draws). For `K = 64` and
`G = 2²⁰`, `Δ ≈ sqrt(20 · 0.69 / 64) ≈ 0.47` — at most one additional
seat per block. Per the §Disambiguation of "K" above, the seat bias
applies to the *selected* committee size; the *signing* attacker share
follows in expectation.

Over `N` consecutive degraded blocks, the bias **compounds
sub-linearly** because each grinding step partially constrains the
state available for the next: an attacker who has biased `N − 1` prior
blocks to retain their share has narrowed the prev_block_hash
distribution available to grind on for block `N`. The accumulated
bias after `N` blocks is approximately:

```
attacker_share(N)  ≈  baseline_share + Δ · sqrt(N)
```

The `sqrt(N)` rather than linear `N` is the saving grace: doubling
the outage length increases the bias by only `sqrt(2) ≈ 1.41×`, not
2×. Concretely, with `baseline_share = 1/3` and `Δ = 0.47/K = 0.0073`:

| Outage duration | Degraded blocks `N` (at 30s blocks) | Worst-case attacker share |
|---|---|---|
| 1 hour | 120 | 33% + 0.7%·sqrt(120) ≈ 41% |
| 6 hours | 720 | 33% + 0.7%·sqrt(720) ≈ 52% |
| 24 hours | 2880 | 33% + 0.7%·sqrt(2880) ≈ 71% |
| 72 hours | 8640 | 33% + 0.7%·sqrt(8640) ≈ 98% |

The takeaway: **drand outages of less than 6 hours leave the attacker
below honest majority; outages of more than 24 hours risk attacker
control of the L2 finality threshold.** This is a quantification, not
a guarantee — the constants `G` and the `sqrt(N)` compounding model
are worst-case estimates calibratable on b2 data. The structure of
the bound (sub-linear compounding, ~24h critical threshold for
realistic adversaries) is robust to the exact constants.

**Operational implication.** A drand outage exceeding
`DRAND_OUTAGE_EMERGENCY_THRESHOLD` (RFC-0008 §7, default **6 hours**)
triggers an emergency-response posture: the L2 committee broadcasts a
notice on the chain; the foundation (Manifesto Thesis 16) acts as
coordinator; new bond admissions and committee role assumptions are
paused until drand resumes or until manual entropy-source substitution
(threshold-BLS committee, VDF) is invoked per the §"BLS12-381
ciphersuite naming has churned" §"What we have not figured out yet"
catalogue. The 6-hour threshold leaves substantial margin against the
24-hour critical-threshold above.

---

## 5 · Embedding model pinning

RFC-0002 requires every peer to compute identical embeddings for identical
inputs. The canonical model is specified as a *pinned checkpoint*, identified
by three values:

```
embedding_model_id        = "ants-embed-v1"
embedding_model_arch      = "bge-m3"            (architecture family)
embedding_model_weights_hash = BLAKE3(canonical_safetensors_file)
embedding_model_tokenizer_hash = BLAKE3(canonical_tokenizer_json)
```

The `embedding_model_weights_hash` and `embedding_model_tokenizer_hash` are
**pinned at the protocol-version boundary**. The specific 32-byte values for
v1 will be set when the reference client is published, computed against the
exact BGE-M3 checkpoint and tokenizer the reference client ships with. Until
then, they are `0x000…000` placeholders.

A peer that wishes to claim `embedding_model_id = "ants-embed-v1"` MUST be
running a model whose safetensors-encoded weights hash to the pinned value and
whose tokenizer JSON hashes to the pinned tokenizer value. There is no
"close enough" — bit-exact match or rejection.

Future versions (`ants-embed-v2`) are introduced via the embedding-model
governance process specified in RFC-0002 v0.2 (forthcoming amendment). That
amendment closes BLOCK B5 from the pre-implementation criticality review.

---

## 6 · NCS encoding

NCS (normalised compute-seconds, RFC-0001) is the unit of community-layer
ledger accounting. Floating-point representation is **forbidden** for ledger
arithmetic — the "0.1 + 0.2 ≠ 0.3" class of bug is unacceptable in an
append-only economic record.

```
NCS is represented as u64 μ-NCS  (micro-NCS, 10^-6 NCS per unit)

1 NCS                = 1_000_000 μNCS
1 μNCS               = 1 unit of the u64
max representable    ≈ 1.8 × 10^13 NCS  (more than the steady-state lifetime
                                          accounting of any single peer)
```

All NCS quantities in CBOR-encoded protocol objects are encoded as CBOR
unsigned integers (major type 0) representing μNCS. Human-facing JSON may
present NCS as a decimal string or float for readability; the underlying
ledger value is always the u64 μNCS.

Overflow is a protocol error and causes the involved transaction to be
rejected. Negative balances are impossible by construction (each peer pair
maintains two unsigned running totals, `served_to` and `served_by`, plus a
signed delta computed on demand).

---

## 7 · Reference constants

The parameter constants scattered across the design RFCs, collected here with
their source, current value, and calibration status.

**Pinned (P):** must be identical across all peers; changing requires a
coordinated protocol-version bump.
**Calibratable (C):** the protocol value adapts based on b2 testnet evidence;
changes within a major version are permitted.

| Constant | Source | Current value | Status |
|---|---|---|---|
| `EMBEDDING_DIM` | RFC-0002 | 1024 | P |
| `SHARD_KEY_BITS` | RFC-0002 | 64 | P |
| `CACHE_HIT_FEE_FRACTION` | RFC-0001 / 0002 | 10% (5–15% band) | C |
| `PRODUCER_ROYALTY_FRACTION` | RFC-0002 | 65% of cache-hit fee (60–70% band) | C |
| `CACHE_REPLICATION_FACTOR` | RFC-0002 | 3 | C |
| `SIMILARITY_THRESHOLD` | RFC-0002 | 0.92 cosine | C |
| `CHOKE_WINDOW` | RFC-0001 | 20 minutes | C |
| `CHOKE_LOOP_INTERVAL` | RFC-0001 | 10 seconds | C |
| `OPTIMISTIC_UNCHOKE_RATIO` | RFC-0001 | 12.5% (1/8 slots) | C |
| `TIER1_SAMPLING_RATE_p` | RFC-0003 | 0.03 (default; derived from b1; the v0.x reference client pins 1/32 via `audit_threshold = 2^59`, §11.8) | C |
| `TIER3_COMMITTEE_SIZE_N` | RFC-0003 | 3 (configurable 3–7) | C |
| `TIER3_CONSENSUS_THRESHOLD` | RFC-0003 | ⌈2N/3⌉ | P |
| `TYPE_I_ERROR_BOUND_α` | RFC-0003 | 1e-6 | C |
| `KAPPA_TENURE_RATE_CAP_κ` | RFC-0004 | TBD on b2 | C |
| `DELTA_A_ACTIVE_DECAY` | RFC-0004 | TBD on b2 (minutes scale) | C |
| `DELTA_T_TENURE_DECAY` | RFC-0004 | TBD on b2 (months scale) | C |
| `MU_0_INT8_CANONICAL` | RFC-0003 / RFC-0009 v0.2 | ≈ 0 (q24 quantization floor) | C |
| `MU_0_FP16_CANONICAL` | RFC-0003 / RFC-0009 v0.2 | TBD on b2 (expected 10⁻⁵ to 10⁻⁴) | C |
| `BOND_DISPUTE_WINDOW` | RFC-0004 v0.3 | 7 days (one week) | C |
| `POUH_BOND_C_COMMITTEE` | RFC-0004 v0.3 §Bonds | TBD on b2 (multiplier per pending finding) | C |
| `BOND_RISK_MULT_CACHE_WRITE` | RFC-0004 v0.3 §Bonds | TBD on b2 (multiplier on est. lifetime royalty) | C |
| `BOND_RISK_MULT_SETTLEMENT` | RFC-0004 v0.3 §Bonds | TBD on b2 (multiplier on settlement value) | C |
| `BLS_TRANSITION_K` | §3.3 above | 16 (Ed25519 ≤ K, BLS > K) | C |
| `DRAND_TIMEOUT` | §4.3 above | 30 seconds | C |
| `ROTATION_ADMISSION_WINDOW` | RFC-0010 §Trustee key rotation | 7 epochs (≈ 1 week) | C |
| `T_FORK_CHOICE_CAP` | RFC-0004 §Fork choice by Σ T_eff | TBD on b2 (≈ 2 years of κ-bound tenure) | C |
| `R_ARCHIVE_MIN` | RFC-0004 §Archive nodes and the pruning-DoS defence | 8 distinct holders per pruned proof | C |
| `EMERGENCY_REVOCATION_WINDOW` | RFC-0010 §Equivocation escape hatch | 30 epochs (≈ 30 days) | C |
| `ATTESTATION_FRESHNESS_WINDOW` | RFC-0005 §Attestation freshness window | 30 days | C |
| `VENDOR_REVOCATION_PROPAGATION_BOUND` | RFC-0005 §Vendor revocation propagation timing | 72 hours | C |
| `DRAND_OUTAGE_EMERGENCY_THRESHOLD` | RFC-0008 §4.3 grinding compounding bound | 6 hours | C |
| `POUH_COMMITTEE_SIZE_K` | RFC-0004 | 64 | C (bootstrap-scales) |
| `POUH_BLOCK_TIME` | RFC-0004 | 30 seconds | C |
| `POUH_FINALITY_THRESHOLD` | RFC-0004 | ⌈2K/3⌉ | P |
| `EPOCH_DURATION` | RFC-0004 | 24 hours | C |
| `MULTI_VENDOR_WEIGHT_SINGLE` | RFC-0005 | 1.0 | C |
| `MULTI_VENDOR_WEIGHT_DUAL` | RFC-0005 | 1.2 | C |
| `MULTI_VENDOR_WEIGHT_TRIPLE` | RFC-0005 | 1.4 | C |
| `DHT_K` | §11.2 (Kademlia bucket size) | 20 | C |
| `DHT_MAX_PEERS_PER_MSG` | §11.2 | 20 (= `DHT_K`; a response carrying more is rejected) | P |
| `DHT_ALPHA` | §11.2 (lookup concurrency) | 3 | C |
| `DHT_BUCKET_COUNT` | §11.2 | 256 (one per XOR-distance bit) | P |
| `GOSSIP_FANOUT_f` | RFC-0004 §The propagation bound; §11.3 | 6 | C |
| `GOSSIP_MAX_FRAME_BYTES` | §11.3 | 4096 (receiver default; local policy) | C |
| `GOSSIP_MAX_IDS_PER_DIGEST` | §11.3 | 64 (an IHAVE/IWANT naming more is malformed) | P |
| `RECEIPT_NONCE_BYTES` | RFC-0004 §The receipt; §11.9 | 16 | P |
| `SUMMARY_MAX_BUCKETS` | RFC-0004 §Compact summaries; §11.9 | 256 (a summary naming more is malformed) | P |
| `SUMMARY_BUCKET_WIDTH` | RFC-0004 §Compact summaries; §11.9 | 86400 s (1 day; DRAFT default — granularity is calibratable) | C |

Pinned values that change require a major version increment. Calibratable
values change within the version via amendment to the corresponding RFC, with
the CHANGELOG entry citing the calibration evidence.

---

## 8 · Test vectors

A separate repository `Ants-Community/ants-test-vectors` hosts versioned
test vectors for every protocol-level primitive. Each vector is a CBOR file
with the structure:

```
TestVector {
  primitive:    string,           // e.g. "blake3.derive_key", "ed25519.sign"
  context:      string,           // domain separation, where applicable
  input:        bytes / structured,
  output:       bytes / structured,
  notes:        text,
}
```

Vector categories (each with multiple instances):

- BLAKE3 plain hash and derive_key per reserved context
- Ed25519 signature generation and verification
- BLS12-381 signature aggregation and verification
- ECVRF prove and verify
- CBOR canonical encoding of every protocol object type
- Merkle tree construction over logit traces (RFC-0003)
- Fault proof verification (RFC-0004)
- Beacon-derived selection (audit-flag, position-set, auditor) for given
  seeds
- Cache entry signing and verification
- Per-position discrepancy score X_j computation
- Betting e-process M_n computation
- Bond formula application per act class
- DHT RPC and gossip frame round-trips (§11.2–§11.3)
- L2 chain objects: epoch summaries, blocks, the confirmed-proofs Merkle
  root and inclusion paths (§11.6)
- Receipt bodies, receipt-bag inclusion proofs, and compact summaries
  (§11.9)

These vectors are the *implementation-level* spec. Any client that produces
different output for the same vector input is non-conformant.

The repository exists (created 2026-05-20, alongside the reference client)
and so far ships packs for BLAKE3, Ed25519, BLS12-381, ECVRF-ELL2, and
canonical CBOR. The early packs are JSON files whose fields mirror the
`TestVector` shape — pragmatic for day-zero review; re-emission as
canonical-CBOR `TestVector` objects per this section is pending and tracked
in the pack README.

---

## 9 · Versioning of this document

This document is **versioned independently** from the design RFCs because it
changes on a different cadence. Updates fall into three categories:

- **Non-breaking additions** (new context strings, new reference vectors, new
  test vector types, new calibratable constants): minor version bump (v0.1 →
  v0.2). All existing peers remain interoperable.
- **Breaking changes to pinned primitives** (replacing BLAKE3, changing the
  embedding model, changing CBOR encoding rules): major version bump (v1 → v2)
  and a coordinated protocol-version event across the network.
- **Constant re-calibrations** (CHOKE_WINDOW, p, μ_0, etc.): no version bump
  to this document; the change is recorded as an amendment to the originating
  design RFC, with a back-reference here.

The breaking-change path requires the same governance discipline as
amendments to the design RFCs: public discussion, BDFL merge during v0.x, TSC
vote post-v1.0. Major version changes are explicitly scope-limited — they do
not bundle unrelated changes.

---

## 10 · What we have not figured out yet

- **The exact BLAKE3 hash of the canonical BGE-M3 checkpoint** is pending
  until the reference client packages a specific version. Today this RFC
  pins the *mechanism*; the *value* is filled in when the client ships.
- **Migrating the superseded derive_key contexts.** §4.1's beacon-challenge
  and Merkle context strings were reserved before the reference
  implementation pinned the keystream (§11.8) and prefix-byte (§11.4)
  constructions instead. Re-pinning those derivations onto `derive_key`
  would be cleaner domain-separation hygiene, but it breaks every committed
  root and challenge — so it happens before first cross-implementation
  interop or not at all.
- **The v0.x reference-model weight blob** (magic `ANTSMOD1`) is pinned in
  the reference client pending an RFC-0009 reference-model appendix. It is
  deliberately *not* specified here: the model is illustrative and swaps
  behind the reference-distribution seam — the protocol's interop surface is
  the commit and the answer envelope (§11.8), not the model.
- **CBOR canonicalisation libraries vary in conformance.** Several CBOR
  libraries do not implement RFC 8949 §4.2.1 deterministic encoding
  correctly. The reference client will validate that its CBOR encoder
  produces byte-identical output against test vectors at startup. We will
  identify and document conformant library versions per language.
- **BLS12-381 ciphersuite naming has churned in the IETF process.** The
  ciphersuite ID may change between draft revisions of the BLS signature
  draft before final RFC publication. We pin to the current value and will
  re-pin when the IETF publishes the RFC.
- **Embedding model upgrade governance** is BLOCK B5 from the
  pre-implementation criticality review; it lives in RFC-0002 v0.2.
- **Reference kernel library for canonical numerics** is BLOCK B1, addressed
  in RFC-0009.

---

## 11 · Protocol message encodings (reference)

*Added in v0.6.* The design RFCs name the protocol's objects; this section
pins their bytes. Everything below was first defined by the reference client
(`ants-client`), module by module, under an explicit "DRAFT pending RFC-0008"
marker; v0.6 promotes those layouts into the specification. From here on the
rule inverts: this document owns the bytes, and the reference client conforms
to it.

These encodings are interoperability-bearing — two implementations that
disagree on any of them cannot exchange the object — but the protocol has not
yet had its first cross-implementation interop event, so changes are still
possible at minor-version cost (§9). After first interop they harden to the
breaking-change path.

Shared conventions, stated once:

- Every object is canonical CBOR per §1.1, definite-length, float-free.
  Maps are integer-keyed (compact, and canonical key order is numeric order,
  so encoders emit fields in declaration order). The DHT alone uses text keys
  in its message *bodies* — see §11.2.
- Hashes are 32-byte BLAKE3 (§2.1); signatures are 64-byte Ed25519 (§3.1)
  unless a section says otherwise. A peer-id is the peer's 32-byte Ed25519
  public key.
- Wire enumerations (message types, fault classes, severities, tiers) are
  **integer-stable and append-only**: a deployed value never changes meaning,
  deprecated values are reserved and never recycled, and an implementation
  that meets a value it does not know fails closed — a typed "don't know",
  never a guess. §11.11 collects the registries.
- A cryptographic check that fails on untrusted input is a **verdict**
  (false), not an error; only a malformed call is an error. Every verifier
  below implements that contract.

### 11.1 · Transport binding

ANTS speaks **QUIC v1** with **ALPN `"ants"`** and mutual-auth TLS 1.3 using
**RFC 7250 raw public keys** on both sides — no X.509, no CA: the TLS-layer
identity *is* the protocol peer-id. The Certificate message carries the
Ed25519 SubjectPublicKeyInfo, a fixed 12-byte DER prefix plus the raw key,
44 bytes total:

```
30 2a                 SEQUENCE (42)
   30 05              SEQUENCE (5)
      06 03 2b 65 70  OID 1.3.101.112 (id-Ed25519)
   03 21 00           BIT STRING (33), 0 unused bits
   <pubkey: 32 bytes>
```

Addresses are textual multiaddrs, libp2p convention:
`/ip4/A.B.C.D/udp/PORT/quic-v1`, with an optional trailing `/p2p/<id>` naming
the expected peer. A dialer that knows whom it expects (bootstrap seeds,
RFC-0010) MUST check the raw public key received in the handshake against
that expectation and abort on mismatch.

Stream discipline, by subsystem:

- **DHT (§11.2):** one RPC per *bidirectional* stream. The requester writes
  one envelope and FINs; the responder writes one envelope and FINs. The FIN
  is the message boundary — no length prefix.
- **Gossip (§11.3):** one persistent *unidirectional* stream per connection,
  opened by the sender and reused for every frame pushed to that peer.
  Because the stream is never FIN'd per message, each frame is
  length-delimited: a 2-byte **big-endian** length prefix precedes the frame
  bytes.

### 11.2 · DHT RPC

Routing is Kademlia over the 256-bit peer-id space: XOR distance, 256 buckets
(one per distance bit), `K = 20` entries per bucket, lookup concurrency
`α = 3`. Shard rendezvous is BitTorrent-style announce/get-peers. A 64-bit
cache shard key (RFC-0002) maps into the address space as
`BLAKE3(shard_key as 8 little-endian bytes)` → the 32-byte lookup target.

Every message is one envelope:

```
envelope = {
  0: type   (uint)   ; 1..8, registry below
  1: txid   (uint)   ; request identifier, echoed verbatim by the response
  2: body   (map)    ; per-type fields, possibly empty
}
```

| type | message | body |
|---|---|---|
| 1 | PING | `{}` |
| 2 | FIND_NODE | `{"target": bytes(32)}` |
| 3 | GET_PEERS | `{"key": uint}` — the 64-bit shard key |
| 4 | ANNOUNCE_PEER | `{"key": uint, "token": bytes(16)}` |
| 5 | PING_RESP | `{}` |
| 6 | FIND_NODE_RESP | `{"peers": [peer, …]}` |
| 7 | GET_PEERS_RESP | `{"peers": [peer, …], "token": bytes(16)}` |
| 8 | ANNOUNCE_PEER_RESP | `{}` |

A peer descriptor is `peer = {"id": bytes(32), "addr": text}` — peer-id plus
multiaddr. (Per §1.1 the shorter key encodes lower, so `"id"` precedes
`"addr"` on the wire.) A response carries at most `K = 20` peers; more is
rejected. FIND_NODE_RESP returns the K closest to `target` by XOR distance;
GET_PEERS_RESP returns announced holders of the shard when it has any,
otherwise the K closest to the shard's expansion so the requester can
iterate.

The `token` is opaque to the requester: 16 bytes minted by the responder,
echoed back in a subsequent ANNOUNCE_PEER for the same shard. It defeats
off-path announce spoofing (you must have received the GET_PEERS_RESP to
know it); how a node mints tokens is local policy, not protocol.

The DHT bodies use text keys while the envelope — and every other object
family in this section — uses integer keys. Both are canonical under §1.1;
the mixed convention stays as deployed, because re-keying a working wire
format is exactly the kind of change §9 prices as breaking.

### 11.3 · Gossip frames

The gossip overlay (RFC-0004 §Layer 1) pushes fault proofs eagerly and fills
gaps by lazy pull. Three frame types, each a `map(2)` with integer keys:

```
PUSH  = { 0: 0, 1: proof (bytes) }             ; one fault-proof envelope (§11.5)
IHAVE = { 0: 1, 1: ids (array of bytes(32)) }  ; "I hold these content-ids"
IWANT = { 0: 2, 1: ids (array of bytes(32)) }  ; "send me these"
```

An IHAVE/IWANT names 1..64 content-ids; more is malformed. Frames above the
receiver's frame cap (default 4096 bytes) are rejected without decoding. The
receive path is the RFC-0004 reference sketch verbatim: VERIFY-insert into
the G-Set; new → forward to up-to-`f` view peers excluding the sender;
duplicate → drop (the epidemic terminates); VERIFY-fail → drop and count
against the sender. An IHAVE is answered with an IWANT naming the ids the
local set lacks; an IWANT is answered with an ordinary PUSH per held id —
both replies go to the requesting peer only.

A late joiner imports a peer's whole set as a **snapshot frame**: a
definite-length `array(N)` whose elements are the N proof byte-strings.
Element order is the sender's internal order — deliberately unspecified,
unsigned, and not content-addressed. The importer re-runs VERIFY on every
element and dedupes by content-id, so the frame needs no authenticity of its
own: the proofs carry theirs.

### 11.4 · The canonical Merkle construction

One Merkle recipe serves every tree in the protocol — prefix-byte domain
separation (RFC 6962 style) over BLAKE3:

```
leaf  = BLAKE3( 0x00 ‖ leaf-preimage )
node  = BLAKE3( 0x01 ‖ left ‖ right )
empty = BLAKE3( 0x02 )            (only where an empty tree is legal)
```

A level with an odd node count **promotes** the lone trailing node unchanged
to the next level — no duplication, so the duplicate-pair ambiguity of
Bitcoin-style trees does not arise, and the prefix bytes close leaf/interior
puns. An inclusion path is the bottom-up sequence of sibling hashes; which
levels carry a sibling versus a promotion is fully determined by
`(index, n_leaves)`, which every verifier replays.

The instantiations:

| Tree | Leaf preimage | Empty tree | Where |
|---|---|---|---|
| `confirmed_proofs` (epoch summary) | the L1 proof's 32-byte content-id; leaves strictly ascending | legal — `BLAKE3(0x02)` | §11.6 |
| Commit-at-send logit trace | `dist_bytes ‖ LE64(position)` | not legal (L ≥ 1) | §11.8 |
| Receipt bag | `body ‖ server_sig ‖ client_sig` | legal — `BLAKE3(0x02)` | §11.9 |

The invalid-transition fault (§11.5) folds a citation against
`confirmed_proofs` with this exact recipe; the reference client pins the two
implementations against each other bit-for-bit in its tests.

This construction supersedes, in the reference encodings, the
`derive_key`-based Merkle contexts reserved by §4.1 in v0.1
(`ants-v1-fault-merkle`, `ants-v1-logit-trace-leaf/-root`); see §10.

### 11.5 · Layer-1 fault proofs

The envelope every fault proof shares (RFC-0004 §Layer 1):

```
fault_proof = {
  0: subject    (bytes 32)  ; the accused peer-id
  1: fault_type (uint)      ; registry in §11.11
  2: epoch      (uint)      ; the contested epoch — what G-Set pruning orders by
  3: evidence               ; class-specific, a NATIVE CBOR value (not a
                            ; byte-string-wrapped sub-document): the proof is
                            ; one canonical document, encoded in a single pass
}
```

`VERIFY(π)` is pure, deterministic, and context-free; the verdict rules for
the two assigned classes are specified in RFC-0004 (v0.7). The
content-address that keys the G-Set is plain `BLAKE3` over the envelope
bytes.

**EQUIVOCATION (fault_type 0).** `evidence = array(2)`, each element an
`array(2) [ body (bytes), sig (bytes 64) ]`: two signed statement bodies. A
statement body is itself canonical CBOR:

```
statement = {
  0: domain  (uint)   ; statement-kind separation — registry in §11.11
  1: epoch   (uint)
  2: slot    (uint)
  3: payload (bytes)  ; what the subject committed to at that slot
}
```

The proof verifies iff both bodies are canonical, share the envelope's epoch
and each other's domain and slot, the payloads **differ**, and both
signatures verify under `subject` over the exact body bytes.

**INVALID_TRANSITION (fault_type 1).** `evidence = array(6)`, elements in
this fixed order:

```
[ block    (bytes)     ; the complete canonical block encoding (§11.6)
  sig      (bytes 64)  ; the subject's Ed25519 over the 32-byte block hash
  cited    (bytes)     ; the committed bytes claimed not to be a valid proof
  index    (uint)      ; the cited leaf's position
  n_leaves (uint)      ; the confirmed set's leaf count (≥ 1)
  path     (bytes)     ; bottom-up siblings, a whole number of 32-byte hashes ]
```

The envelope's `epoch` MUST equal the block summary's epoch (pruning
alignment). The verdict rules — including the no-nesting bound and the
decode-failure-is-permanent invariant — are RFC-0004 v0.7 §"Fabrication is
attributable". The signature is over the *naked* 32-byte block hash, the
exact message committee attestations sign (§11.6): that identity is what
makes fabrication signer-attributable with no committee or finality context.

### 11.6 · Layer-2 chain objects

**Epoch summary** — the signable core; attestations ride on the block:

```
epoch_summary = {
  1: epoch            (uint)
  2: cutoff_time      (uint)      ; unix seconds; the L1 view is "as of" this T
  3: confirmed_proofs (bytes 32)  ; §11.4 root over the visible proofs' content-ids
  4: findings         (array of finding)
}
finding = {
  1: subject  (bytes 32)
  2: rule_id  (uint)
  3: window_s (uint)
  4: count    (uint)
  5: severity (uint)              ; registry in §11.11
}
```

**Block:**

```
block = {
  1: height          (uint)
  2: prev_block_hash (bytes 32)
  3: degraded_seed   (bool)       ; §4.3 — INSIDE the signed bytes: the
                                  ; committee attests to the degradation claim
  4: summary         (bytes)      ; the epoch_summary encoding, as a byte string
}
```

`block_hash = BLAKE3(block encoding)`. A committee attestation is an Ed25519
signature over that 32-byte hash. Finality is at least `⌈2K/3⌉` valid
attestations, counted against the VRF-selected `K` (§3.3 disambiguation). At
v0.x every attestation is per-signer Ed25519 regardless of K; the BLS
aggregate above `BLS_TRANSITION_K` (§3.3) is a deferred wire compression,
not a different trust model.

Proposal is deterministic. The proposer at `height` is committee position
`height mod K` — a DRAFT rule: RFC-0004 requires only "deterministic within
the committee", and the committee *set* is already beacon-seeded, so the
position adds no grinding surface. The candidate block is a pure function of
the proposer's L1 view at `cutoff_time` — the pattern engine runs with
`now ≡ cutoff_time`, never wall clock — so any member recomputes the
byte-identical block from the same view, and recompute-and-compare *is*
proposal validation (members whose view lags abstain; 2/3 absorbs the
stragglers).

### 11.7 · External beacon (drand)

The protocol consumes drand's **default chain** (scheme
`pedersen-bls-chained`), which is exactly the protocol's own BLS shape
(§3.3): 48-byte compressed G1 group key, 96-byte G2 signatures, the same
ciphersuite. One verified round:

```
round_message = SHA-256( previous_signature ‖ round_be64 )
verify        : BLS_verify(group_key, round_message, signature)
randomness    = SHA-256( signature )       ; must equal the published value
```

SHA-256 here is drand's protocol, accepted as-is per §2.2; the 32-byte
`randomness` is the round's canonical output and the `drand_round_value` of
§4.2. Rounds below 2 are out of scope (round 1 of a chained drand network
signs the network's genesis seed, a different payload shape; an ANTS genesis
pins `initial_round` well past it, RFC-0010). Which round feeds which block
is the §4.3 rule: the round whose published time is closest to but not after
the proposal timestamp, with the 30-second hard floor before degrading.

### 11.8 · Commit-at-send, challenge derivation, answer envelope

**The commit** (RFC-0003 §commit-at-send) — what a producer signs alongside
every answer:

```
commit = {
  1:  root            (bytes 32)  ; §11.4 root over the per-position leaves
  2:  model_hash      (bytes 32)  ; BLAKE3 of the model artifact
  3:  input_hash      (bytes 32)  ; BLAKE3 of the request input
  4:  req_nonce       (bytes 32)  ; requester freshness
  5:  agent_meas      (bytes 32)  ; TEE measurement (all-zero pending #3, v2.x)
  6:  round           (uint)      ; PoUH round r; the challenge uses B_{r+1}
  7:  audit_threshold (uint)      ; floor(p · 2^64) — the audit probability as
                                  ; fixed-point, keeping the wire float-free
  8:  m               (uint)      ; positions an audit opens
  9:  length          (uint)      ; committed positions L
  10: tier            (uint)      ; 1..3, registry in §11.11
}
```

The signature is Ed25519 over the canonical encoding. A Merkle leaf's
preimage is the position's q24 logit vector serialized little-endian
(vocab × 4 bytes) followed by `LE64(position)` (§11.4); binding the position
into the leaf closes relocation attacks.

**Challenge derivation.** All three audit decisions derive from the
posterior beacon `B_{r+1}` and the commit root through one keystream:

```
seed      = beacon(32) ‖ root(32) ‖ tag    ; tag ∈ {"sel","pos","aud"} — 3 bytes, no NUL
block(i)  = BLAKE3( seed ‖ LE64(i) )       ; i = 0, 1, 2, …
keystream = block(0) ‖ block(1) ‖ …
W_k       = keystream bytes [8k, 8k+8), read BIG-endian
```

- **audited?** ⇔ `W_0 < audit_threshold` (tag `"sel"`).
- **positions:** `M = min(m, L)` contiguous buckets `[⌊jL/M⌋, ⌊(j+1)L/M⌋)`;
  position j = bucket start + `(W_j mod bucket width)` (tag `"pos"`) —
  stratified, distinct, strictly ascending.
- **auditor:** an unbiased index in `[0, n_verifiers)` by rejection sampling
  over successive words (tag `"aud"`) — no modulo bias.

This keystream supersedes the `ants-v1-beacon-*` derive_key contexts
reserved by §4.1 in v0.1 (see §10).

**The answer envelope** — what an auditor opens:

```
answer = {
  0: input  (bytes)              ; the request x
  1: tokens (bytes)              ; the L generated tokens, one byte each
  2: dists  (array(L) of bytes)  ; per-position q24 logit vectors, vocab × 4 bytes LE
}
```

The vocab is implied (`dist length / 4`, ≤ 256): a byte vocabulary, so
`tokens` is unambiguous and the prefix at position j is
`input ‖ tokens[0..j)`. The audit's integrity gate recomputes the §11.4 root
over ALL stored dists and requires it to equal `commit.root` before any
position is scored — one check binds every opening. Reference audit policy:
Tier 1 commits `audit_threshold = 2^59` (p = 1/32); Tiers 2–3 commit
`UINT64_MAX` (always auditable on request).

The reference model whose weights produce these distributions is pinned in
the reference client (magic `ANTSMOD1`) pending an RFC-0009 appendix — see
§10. The protocol's interop surface is the commit and the envelope, not the
model.

### 11.9 · Receipts and selective disclosure

**The receipt body** — the bytes both parties sign (RFC-0004 §"The receipt
that matters for tenure"):

```
receipt_body = {
  1: server    (bytes 32)   ; who did the work — accrues A and T
  2: client    (bytes 32)   ; who countersigns
  3: ncs_value (uint)       ; μ-NCS (§6)
  4: timestamp (uint)       ; unix seconds
  5: nonce     (bytes 16)   ; issuer-fresh anti-replay
}
```

A receipt is the body plus `server_sig` and `client_sig`, both Ed25519 over
the body bytes; a receipt without both valid signatures does not count.

**The receipt bag** commits a peer's history for selective disclosure
(RFC-0004 §Selective disclosure): leaves per §11.4 with preimage
`body ‖ server_sig ‖ client_sig` — the whole receipt, so the inclusion proof
and the countersignature check bind the same bytes. Canonical leaf order is
**timestamp ascending, ties by leaf-hash ascending** — total and
deterministic; duplicates are non-canonical. The published commitment binds
the tree to its peer:

```
bag_root = BLAKE3.derive_key("ants-v1-receipt-bag-root", peer_id ‖ merkle_root)
```

An `A ≥ b` proof is a set of openings `(receipt, index, path)`. The verifier
accepts iff every opening countersignature-verifies, credits this peer, is
not future-dated, sits at a *distinct* index, and Merkle-verifies against
`bag_root` — and the openings' decayed-A contributions (the exact recipe of
RFC-0004's A) sum to at least `b`. Sound as a lower bound: a revealed subset
understates, never overstates.

**The compact summary** (RFC-0004 §Compact summaries for large peers) is the
signed bucket hint a large peer publishes to cut per-admission verification
cost. The signed body:

```
summary_body = {
  1: peer_id      (bytes 32)   ; whose bag — also the signature's verify key
  2: bag_root     (bytes 32)   ; the bag commitment this summary binds
  3: eval_time    (uint)       ; unix seconds — the decay evaluation time
  4: bucket_width (uint)       ; seconds, > 0 (SUMMARY_BUCKET_WIDTH default)
  5: buckets      (array of [bucket_index (uint), total (uint)] pairs)
}
```

The full wire object re-keys the same five pairs as a 6-pair map and adds
`6: sig (bytes 64)` — the peer's Ed25519 signature over the body bytes.
Single-signed: the summary is the peer's own statement about its own bag.

Semantics and canonical form:

- `bucket_index = timestamp / bucket_width` (integer division); `total` is
  the Σ of that bucket's receipts' decayed-A contributions (RFC-0004's exact
  A recipe), evaluated **at `eval_time`**, under exactly A's eligibility
  filter: both signatures valid, credits this peer, not dated after
  `eval_time`.
- Strictly ascending `bucket_index`, no zero totals (an empty bucket is
  OMITTED), at most `SUMMARY_MAX_BUCKETS` buckets — one encoding per
  content, the same discipline as the bag's canonical leaf order.
- RFC-0004's "committed via the same Merkle root" is realised by key 2: the
  signed body *includes* `bag_root`. (A summary leaf inside the tree would
  make the root self-referential.)
- Totals are **not** re-decayable to a later time: the fixed-point decay
  recipe is a pinned repeated multiplication and is not associative, so
  `factor(a+b)` ≠ `fp_mul(factor(a), factor(b))` in general. Audits
  recompute at the summary's own `eval_time`, bit-exactly; a consumer that
  wants fresher numbers asks the peer for a fresher summary.

The summary is a **hint**, not a security primitive: no A or T computation
consumes it. The random audit (RFC-0004: "the verifier may still require
selective disclosure of individual receipts to confirm specific buckets")
names a bucket the summary *states*; the peer answers with openings from
that bucket; the audit passes iff every opening verifies as for `A ≥ b`,
additionally falls in the audited bucket and is not dated after
`eval_time`, and the contributions at `eval_time` sum to **at least** the
stated total. Exceeding the stated total merely means the summary
understated — the safe direction. A signed summary stating a bucket its
receipts cannot substantiate is equivocation against the peer's own
signature — an attributable fault (RFC-0004).

### 11.10 · Bond admission

The admission statement a verifier signs when admitting a bond (RFC-0004
§Bond accounting):

```
bond_admission = {
  1: act_id       (bytes 32)
  2: peer         (bytes 32)   ; whose A is being bonded
  3: amount       (uint)       ; μ-NCS
  4: v_id         (bytes 32)   ; the admitting verifier
  5: l1_view_hash (bytes 32)   ; the verifier's L1 view at admission
}
```

Race-safe tie-break (RFC-0004 §Atomicity): when two verifiers admit
conflicting bonds for the same peer within δ_admission, the admission with
the **smaller** (bytewise)

```
tiebreak_key = BLAKE3.derive_key("ants-v1-bond-tiebreak",
                                 epoch_seed ‖ act_id ‖ v_id ‖ LE64(admission_round))
```

wins, and the loser gossips a null admission. The epoch seed comes from the
L2 chain, so favourable timings cannot be precomputed; only loose
epoch-level clock agreement is required.

### 11.11 · Registries

All append-only; unknown values fail closed.

- **Fault classes** (§11.5): 0 EQUIVOCATION · 1 INVALID_TRANSITION ·
  ≥ 2 reserved.
- **Statement domains** (§11.5): none assigned yet. The registry exists so
  equivocation domains are spec-assigned, never ad-hoc. (The block
  attestation deliberately signs the naked block hash, *not* a statement
  envelope — the invalid-transition fault depends on that exact message
  identity.)
- **Finding severities** (§11.6): 1 SOFT · 2 MEDIUM · 3 HARD · 4 TERMINAL ·
  ≥ 5 reserved.
- **Gossip frame types** (§11.3): 0 PUSH · 1 IHAVE · 2 IWANT.
- **DHT message types** (§11.2): 1–8 as tabled.
- **Verification tiers** (§11.8): 1 attestation+sampling · 2 cross-check ·
  3 triangulation.

Deliberately *not* specified here: the **local ledger record**
(economy/ledger) — it is private per RFC-0001, never crosses the wire, and
acquires a spec section only if record interchange ever becomes a protocol
feature; and the **reference-model weight blob** (§10, RFC-0009's appendix).

---

## 12 · How to contribute

Open an issue on `Ants-Community/ants` with the tag `rfc-0008`. This document
welcomes two kinds of contribution:

- **Conformance findings.** Test results showing that a specific library or
  language produces non-conforming output for a documented primitive — these
  go straight to the spec or to the library's bug tracker.
- **Primitive replacements.** Proposals to change a pinned primitive
  (e.g. "BLAKE3 has a known weakness; we should move to X") — these go
  through the major-version-bump path with the full coordinated upgrade
  discipline.

Routine optimisation suggestions ("we should encode this as Y for 5% smaller
payload") are appreciated but go straight to the design RFC that owns the
object, not here. This document records *which* primitives are used; it does
not own *all* protocol-object designs.

The document is CC0. Adopt it for your own protocol if you find it useful.

---

*Drafted by the ANTS founding contributors, May 2026.
The bytes do not argue. They either match the spec, or they do not.*
