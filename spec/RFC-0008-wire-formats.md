# RFC-0008 — Wire Formats, Crypto Primitives, and Reference Constants

**Status:** Reference · v0.3 (living standard)
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

New contexts are added to this table via amendment to this RFC, never
hard-coded ad-hoc. The domain-separation discipline is non-negotiable: re-using
a derived key across contexts is the single most common cause of subtle
cryptographic bugs in deployed systems.

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
block_height_be64 ‖ drand_round_value)`. drand's threshold-BLS construction
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
| `PRODUCER_ROYALTY_FRACTION` | RFC-0002 | 50% of cache-hit fee | C |
| `CACHE_REPLICATION_FACTOR` | RFC-0002 | 3 | C |
| `SIMILARITY_THRESHOLD` | RFC-0002 | 0.92 cosine | C |
| `CHOKE_WINDOW` | RFC-0001 | 20 minutes | C |
| `CHOKE_LOOP_INTERVAL` | RFC-0001 | 10 seconds | C |
| `OPTIMISTIC_UNCHOKE_RATIO` | RFC-0001 | 12.5% (1/8 slots) | C |
| `TIER1_SAMPLING_RATE_p` | RFC-0003 | 0.03 (default; derived from b1) | C |
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
| `POUH_COMMITTEE_SIZE_K` | RFC-0004 | 64 | C (bootstrap-scales) |
| `POUH_BLOCK_TIME` | RFC-0004 | 30 seconds | C |
| `POUH_FINALITY_THRESHOLD` | RFC-0004 | ⌈2K/3⌉ | P |
| `EPOCH_DURATION` | RFC-0004 | 24 hours | C |
| `MULTI_VENDOR_WEIGHT_SINGLE` | RFC-0005 | 1.0 | C |
| `MULTI_VENDOR_WEIGHT_DUAL` | RFC-0005 | 1.2 | C |
| `MULTI_VENDOR_WEIGHT_TRIPLE` | RFC-0005 | 1.4 | C |

Pinned values that change require a major version increment. Calibratable
values change within the version via amendment to the corresponding RFC, with
the CHANGELOG entry citing the calibration evidence.

---

## 8 · Test vectors

A separate repository `Ants-Community/ants-test-vectors` will host versioned
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

These vectors are the *implementation-level* spec. Any client that produces
different output for the same vector input is non-conformant.

The repository is created when the reference client begins. Until then, this
RFC declares the structure; the population is future work.

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
- **drand round mapping.** The exact convention for which drand round value
  feeds which ANTS chain block. Probably "the drand round whose published
  time is closest to but not after the block-proposal timestamp", but the
  precise rule needs writing.
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

## 11 · How to contribute

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
