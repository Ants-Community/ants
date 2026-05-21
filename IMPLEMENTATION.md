# Implementation Roadmap

*How the all-in-one reference client decomposes into ~15 sub-components, what each costs, what depends on what, and what a realistic timeline looks like with a realistic team.*

---

## Why this document exists

The external multi-persona review of 2026-05-20 made one criticism more
sharply than any other:

> *"The corpus is written as if 'the reference client' were a singular
> monolithic implementation. In practice, the natural decomposition is
> ~12–15 sub-components with different dependencies (transport, identity,
> cache, ledger, CRDT gossip, chain, BLS aggregation, drand integration,
> canonical kernels, etc.). Each is 2–6 months of work engineer-full-time
> for production-grade. The 'all-in-one client' framing hides this
> complexity rather than exposing it — there is no implementation roadmap
> document decomposing the client into independently orderable
> deliverables."*

The criticism is correct. The GOVERNANCE.md "all-in-one client first"
strategy is sound (it forces protocol coherence and avoids partial
implementations) but its absence of a decomposition document gives
prospective contributors no surface to engage with.

This is that document. It is **not an RFC** — it specifies no protocol
behaviour. It is project management. It is intended to grow as the
implementation progresses and to be amended when reality diverges from
plan.

The earlier corpus estimate of "6–12 months with 2–4 engineers" was the
reviewer's correctly-identified 3× underestimate. The honest figure,
re-derived below from per-component bottoms-up estimation, is **24–36
months with 4–6 engineers**.

---

## Language and stack

The reference client is implemented in **C (C99/C11)**. The choice
reflects the project's two operational priorities: maximum performance
on the audited path, and total cross-platform reach (every server,
every Apple Silicon, every Android NDK target, every embedded ARM
chip with a TEE). C is the language that ships on every platform that
matters, has the most mature crypto and ML library ecosystem, has the
most stable ABI of any compiled language, and is the most
audit-friendly because every line does one thing.

The trade-off accepted: no compile-time memory-safety guarantee.
Mitigation through discipline (defer/cleanup macros, arena allocators
for hot paths, static analysis with Clang's `-Weverything`, fuzzing
via libFuzzer/AFL++, ThreadSanitizer + AddressSanitizer in CI). The
manifesto's Thesis 18 acknowledgment of hardware-vendor trust extends
to a comparable register here: we accept the memory-safety cost as
the price of running on every platform at full speed.

**Reference library baselines** (subject to revision; these are the
production-grade C codebases the reference client is expected to
link against or fork):

| Layer | Library |
|---|---|
| Inference primitives | [`ggml`](https://github.com/ggerganov/ggml) — quantized inference in C, foundation underlying `llama.cpp` |
| BLAKE3 | [`BLAKE3-team/BLAKE3`](https://github.com/BLAKE3-team/BLAKE3) C reference implementation |
| Ed25519 | `ed25519-donna` or a formally-verified C implementation |
| BLS12-381 | [`supranational/blst`](https://github.com/supranational/blst) — C + asm, fastest BLS implementation |
| ECVRF (ELL2) | Forked from [`RFC 9381`](https://www.rfc-editor.org/rfc/rfc9381) reference + Elligator 2 C port |
| CBOR | [`tinycbor`](https://github.com/intel/tinycbor) or [`nanocbor`](https://github.com/bergzand/NanoCBOR) — both audited for RFC 8949 §4.2.1 deterministic encoding |
| P2P transport | [`picoquic`](https://github.com/private-octopus/picoquic) — IETF QUIC reference implementation in pure C99, BSD-3-licensed. ANTS-native protocol (DHT, gossip, peer routing) is built on top of picoquic streams. See decision rationale in the component #4 row below. |
| TEE SDK | Intel SGX/TDX SDK, AMD SEV-SNP SDK, ARM CCA reference, Apple SE Objective-C bridge, Qualcomm QSEE — all C/C++ native, no binding work |
| Build system | CMake (cross-platform superbuild) or pure Make + autoconf |

**Repo layout** (per the language-alignment decision, Round 5):
- [`Ants-Community/ants`](https://github.com/Ants-Community/ants) —
  this repo, holds the spec, governance, manifesto, and
  CHANGELOG. No client code.
- `Ants-Community/ants-client` (new, to be created) — CMake
  superbuild of the reference client. One subdir per component
  group (foundation/, network/, reputation/, cache/, inference/,
  economy/). Each component is a static library, all linked into
  the all-in-one client binary.
- `Ants-Community/ants-test-vectors` (new, to be created) —
  versioned test-vector pack per RFC-0008 §8, grows alongside the
  reference client.

**Licence** for the code (per Round 5 decision): **Apache-2.0 OR
MIT** dual-licence. The Apache-2.0 patent grant is non-negotiable
for a crypto/TEE-touching system; the MIT alternative maximises
permissiveness for downstream forks. CC0 was considered (for
manifesto consistency) but rejected because it provides no patent
grant, which is the wrong trade for the code layer specifically.

---

## The fifteen sub-components

Grouped into six layers by architectural concern. Effort estimates are
engineer-months at production-grade quality (not throwaway-prototype
quality).

### Foundation layer (3 components, ~9 engineer-months)

These have no internal dependencies; can be built in parallel from day
one.

| # | Component | Spec | Effort | Notes |
|---|---|---|---|---|
| 1 | Crypto primitives library | RFC-0008 §2–4 | 2 EM | BLAKE3 (C ref impl), Ed25519 (ed25519-donna or verified C), BLS12-381 (blst, C+asm), ECVRF-ELL2 (forked from RFC 9381 reference). Mature C libraries exist; the work is wrapping + test-vector conformance + the ELL2 hash-to-curve port. |
| 2 | CBOR canonical codec | RFC-0008 §1.1 | 1 EM | Deterministic encoding per RFC 8949 §4.2.1. Candidate baselines: `tinycbor` (Intel) or `nanocbor`. Most existing libraries are non-conformant on the deterministic-encoding rules; expect to fork one and tighten it. |
| 3 | TEE attestation harness | RFC-0005 | 6 EM | **Targeted v2.x, not v1.0.** Five vendor families (Intel TDX, AMD SEV-SNP, ARM CCA, Apple SE, Qualcomm QSEE) — all native C/C++ SDKs. Each has its own attestation format + signing chain + revocation flow. Component must also enforce `ATTESTATION_FRESHNESS_WINDOW` (RFC-0005 §Attestation freshness, default 30d) uniformly. **Deferred to v2.x** per RFC-0005's hardware-trust timeline (current silicon-generation vulnerability windows close around 2030-2032; TEE attestation as a participation path becomes operationally meaningful then). v1.0 ships with the API surface as a `NOT_IMPLEMENTED` stub so upstream components can integrate against it; verifiability in v1.0 rests on the other three legs of RFC-0002 (re-execution, scheme (C) probabilistic, reputation). |

### Network layer (3 components, ~9 engineer-months)

Foundation must be complete; layers can then build in parallel.

| # | Component | Spec | Effort | Notes |
|---|---|---|---|---|
| 4 | P2P transport | RFC-0002 §DHT routing, RFC-0010 §First peer's flow | 5 EM | **Stack: vendored [`picoquic`](https://github.com/private-octopus/picoquic) (IETF QUIC reference, pure C99, BSD-3) + ANTS-native protocol layered on QUIC streams.** The earlier "libp2p (3 EM) or custom (6 EM)" framing was abandoned during foundation/network design (2026-05-21): libp2p has no mature C99 implementation (the canonical ones are Go/Rust/JS, and a C++ binding violates the project's C-pure portability discipline). Picoquic gives us IETF-standard QUIC (TLS 1.3 + multiplexed streams + congestion + NAT-friendly handshake) as a ~50k-LOC snapshot-vendorable BSD-3 dependency; the application protocol (peer routing, DHT queries, gossip frames) is written in ANTS-native C99 on top of picoquic streams, not borrowed from libp2p's protocol stack. Ed25519 peer identity binds to the QUIC TLS handshake via raw-public-key (RFC 7250) so we don't carry X.509+CA complexity. |
| 5 | Kademlia DHT (shard-key variant) | RFC-0002 §DHT routing | 3 EM | Standard Kademlia adapted for LSH shard-keys. Mature reference implementations exist. |
| 6 | Gossip overlay (L1 CRDT propagation) | RFC-0004 §Layer 1 + §G-Set pruning | 3 EM | Anti-eclipse peer selection (RFC-0005); gossip discipline; rate limits. |

### Reputation & identity layer (3 components, ~13 engineer-months)

Substantial; touches the architecture's load-bearing layer.

| # | Component | Spec | Effort | Notes |
|---|---|---|---|---|
| 7 | L1 CRDT (G-Set) implementation | RFC-0004 v0.5 §Layer 1 incl. pruning | 4 EM | VERIFY function for all proof types, gossip integration, pruning policy, late-joiner protocol, archive-node interface. |
| 8 | L2 PoUH chain | RFC-0004 v0.5 §Layer 2 incl. partition recovery | 6 EM | Block production, BLS aggregation (K>16) + Ed25519 multi-sig (K≤16), VRF committee selection, epoch summary generation, drand integration with failover, equivocation slashing, Σ-T fork choice. **The single longest implementation deliverable in this layer.** |
| 9 | Identity & reputation service | RFC-0004 v0.5 §Tenure + §Bond accounting + §Selective disclosure; RFC-0005 | 3 EM | Receipt bag Merkle tree, (A, T, κ) computation, bond accounting with race-safe admission, selective-disclosure protocol, multi-vendor attestation lookups. |

### Cache layer (2 components, ~6 engineer-months)

| # | Component | Spec | Effort | Notes |
|---|---|---|---|---|
| 10 | Semantic cache (DHT-routed) | RFC-0002 v0.2 | 5 EM | LSH shard-keys, FAISS-backed local shards, replication factor, eviction policy, write protocol, lookup protocol, cross-economy royalty accounting. |
| 11 | Canonical embedding service | RFC-0002 v0.2 §The canonical embedding model + RFC-0008 §5 | 1 EM | BGE-M3 inference with hash verification at startup. Mostly packaging an existing model. |

### Inference layer (2 components, ~10–18 engineer-months)

The economic engine; also the longest-tail effort.

| # | Component | Spec | Effort | Notes |
|---|---|---|---|---|
| 12 | Canonical kernel library (`ants-canonical-kernels`) | RFC-0009 v0.5 | 6–12 EM | C library, expected to leverage `ggml` as computational foundation with the discipline layer (bit-exact reductions per §2.1, INT8-GPTQ-128, q24 conversion) above it. CPU INT8 reference kernels for x86 AVX2/AVX-512 + ARM NEON/SVE, FP16 fallback. GPU canonical kernels are explicitly *future work* per RFC-0009 and add another 6–18 EM each (per-platform: CUDA, ROCm, MPS). **The largest engineering risk in the entire corpus.** |
| 13 | Inference orchestration | RFC-0003 v0.2, RFC-0009 v0.2 | 4 EM | Model loading, request routing, fast-path serving (the producer's chosen kernel), audited-path commit (commit-at-send with Merkle binding), e-process audit verifier role, Tier 1/2/3 dispatch. |

### Economy & coordination layer (2 components, ~4 engineer-months)

The user-visible economic surface.

| # | Component | Spec | Effort | Notes |
|---|---|---|---|---|
| 14 | Local economic ledger | RFC-0001 v0.3 | 2 EM | Per-pair NCS accounting in u64 μNCS (RFC-0008 §6), choke/unchoke loop, optimistic-unchoke slot, payment-terms negotiation per RFC-0006. |
| 15 | Bond accounting service | RFC-0004 v0.5 §Bond accounting model + §Atomicity | 2 EM | A computation, locked bonds tracking, race-safe admission with δ_admission + BLAKE3 tie-break, integration with verifier roles in RFC-0003 (Tier 3) and RFC-0004 (PoUH committee). |

### Sum

| Layer | Components | Engineer-months (low) | Engineer-months (high) |
|---|---|---|---|
| Foundation | 3 | 9 | 9 |
| Network | 3 | 9 | 12 |
| Reputation & identity | 3 | 13 | 13 |
| Cache | 2 | 6 | 6 |
| Inference | 2 | 10 | 18 |
| Economy & coordination | 2 | 4 | 4 |
| **Sum (component work)** | **15** | **51** | **62** |
| Integration testing | — | +15 | +20 |
| Security audit & response | — | +6 | +10 |
| **Total** | — | **72** | **92** |

72–92 engineer-months. Divided by team size:
- **4 engineers full-time**: 18–23 months (best case, perfect parallelisation)
- **5 engineers full-time**: 14–18 months
- **6 engineers full-time**: 12–15 months

Add 30–50% for the inherent friction of distributed-systems development
under adversarial threat model (this is not a typical CRUD app):

- **4 engineers**: 24–34 months
- **6 engineers**: 16–22 months

**Realistic target: 24–36 months with a team of 4–6 engineers.** This is
the figure to communicate to prospective contributors and funders. The
6–12 months with 2–4 figure that earlier corpus references mentioned was
~3× underestimation.

---

## Critical path

The longest-dependency chain through the components:

```
Foundation (parallel: #1, #2, #3)
   │
   └──> Network layer (parallel: #4 depends on #1; #5 depends on #4; #6 depends on #4+#1+#2)
            │
            └──> Reputation+Identity (parallel: #7 depends on #6; #8 depends on #7+#1; #9 depends on #1+#2+#7)
                     │
                     ├──> Cache (#10 depends on #4+#5+#9; #11 depends on #1)
                     │       │
                     │       └──> Inference (#12 independent of network; #13 depends on #11+#12+#7+#9+#10)
                     │                │
                     │                └──> Economy (#14 depends on #9; #15 depends on #9+#13)
                     │
                     └──> Integration testing once all reach feature-complete
```

The single longest unbreakable chain for **v1.0** is approximately:

`#7 (L1 CRDT, 4 EM) → #8 (L2 chain, 6 EM) → #13 (inference orchestration, 4 EM) → #15 (bond accounting, 2 EM)`

= 16 engineer-months on the critical path with all other components
running in parallel. Even with infinite parallel engineering the
wall-clock minimum is ~16 months from start to feature-complete, plus
integration testing.

(Component **#3 (TEE attestation harness)** is no longer on the v1.0
critical path: per RFC-0005's hardware-trust timeline it is targeted
for v2.x (2030-2032 silicon-vulnerability window closure). v1.0 ships
with a `NOT_IMPLEMENTED` stub for the TEE API surface; verifiability
rests on the other three legs of RFC-0002.)

The component most likely to slip: **#12 (canonical kernel library)** if
GPU canonical kernels turn out to be intractable on key platforms.
Mitigation: ship with CPU-only canonical (acceptable for v0.1 testnet),
add GPU canonical kernels per platform as v0.2+ deliverables.

---

## Six-phase plan (months 1–36, conservative team of 4)

| Phase | Months | Focus | Deliverable |
|---|---|---|---|
| **A** | 1–6 | Foundation + start canonical kernel | Crypto + CBOR done; TEE harness `NOT_IMPLEMENTED` stub in place (real implementation deferred to v2.x per RFC-0005); canonical CPU kernel for one architecture; P2P transport via vendored picoquic + ANTS-native protocol on top. |
| **B** | 6–12 | Finish network layer + start reputation/identity + start cache | DHT + gossip done; L1 CRDT functional (no pruning yet); L2 chain skeleton with Ed25519 multi-sig; identity service first version; cache LSH routing working. |
| **C** | 12–18 | Finish reputation/identity + start inference orchestration + finish cache | L1 CRDT with pruning + late-joiner; L2 chain with BLS transition + partition recovery; identity full; bond accounting first version; inference orchestration skeleton. |
| **D** | 18–24 | Finish inference layer + economy + coordination + first integration | Canonical kernels for all CPU architectures; FP16 fallback; inference orchestration with commit-at-send + e-process audit; local ledger + bond accounting integrated. |
| **E** | 24–30 | Integration testing + first internal testnet | All components integrated; testnet running with founding-trustee bootstrap committee; capability matrix phase 1–2 (per RFC-0010); first b2-style measurements of `e`, `μ_0`, `T_prop`. |
| **F** | 30–36 | External security audit + bug fixes + performance optimisation + mainnet preparation | Audit findings addressed; performance benchmarks (latency, throughput, verifiability tax); mainnet GenesisState assembled per RFC-0010; v0.1 release candidate. |

A team of 6 engineers compresses this to 18–24 months wall-clock with a
similar phasing.

---

## Parallelisable vs critical-path work

For a team-of-4 with limited bandwidth, the work that can be paused
without slipping the critical path:

**Critical path (must not slip):**
- L1 CRDT + L2 chain (and their composition)
- Canonical kernel library
- Integration testing
- (TEE attestation harness is **no longer on the v1.0 critical path** —
  see component #3 above and the v2.x deferral notes throughout.)

**Parallel work (can slip without delaying the critical path):**
- Cache layer (depends on reputation but doesn't block it)
- Economy & coordination (depends on reputation but doesn't block it)
- Canonical embedding service (small, mostly packaging)
- Test vectors (`ants-test-vectors` repository — should grow in parallel
  with the reference client)

If a contributor wants to start with something small to learn the codebase,
**components #2 (CBOR codec), #11 (embedding service), #15 (bond accounting
service)** are good first contributions — each is 1–2 EM, has clear spec
references, and tests cleanly against test vectors.

---

## Test vector pack

A sibling repository `Ants-Community/ants-test-vectors` must grow in
parallel with the reference client. Per RFC-0008 §8, every primitive
should have at least one input/output test vector. The reference client's
implementation is *conformant* iff its outputs match every vector. The
vectors themselves are generated by the reference client (chicken-and-egg
acknowledged) and versioned alongside it.

A practical workflow:
- Reference client is the *first* implementation.
- Each component, on feature-complete, generates its test vectors.
- The vectors get committed to `ants-test-vectors`.
- Future client implementations (per the v1.0 condition of 3+ independent
  implementations) validate against the vectors.

A non-trivial contribution opportunity: building the test-vector
generation tooling and the conformance test runner.

---

## How to claim a sub-component

This is a tentative process; will be amended as the project grows.

1. **Open an issue** on `Ants-Community/ants` titled
   `[CLAIM] Component #N — [name]`.
2. **Describe**: your background, why you're suited, expected start
   date, expected duration, working hours per week.
3. **Discussion**: the BDFL (v0.x) or TSC (post-v1.0) reviews. Multiple
   claims for the same component go to the most-qualified-and-available;
   parallelisable sub-tasks within a component may be shared.
4. **Acknowledgement**: the claim is recorded in
   `CONTRIBUTORS.md` (to be created), and you become the primary
   contributor for that component.
5. **Drift**: a primary contributor who falls silent for >2 months
   without communication may have their claim reassigned by the
   maintainer process.

For now, the implicit primary on every component is "the BDFL or
delegate, pending claim". Claims are highly welcome.

---

## What this document is not

- **It is not a contract.** Estimates are estimates; reality will diverge.
  The honest framing is "this is the size of the box."
- **It is not a hiring document.** Engineers paid to do this work would
  have different productivity than volunteer contributors. Both kinds of
  contribution are welcome.
- **It is not the project's only success path.** Components might be
  combined, split, swapped (e.g., a future contributor proposes a
  superior canonical-kernel approach). The decomposition is a working
  scaffold, not a constraint.

---

## How to amend this document

Open a PR with the proposed change and a rationale. Substantive changes
(component additions/removals/major effort revisions) follow the
BDFL/TSC review per GOVERNANCE.md. Minor changes (typo, clarity, more
accurate estimate based on completed component) are merged quickly.

When a component is completed, mark it `COMPLETE` here with the actual
effort spent, so future estimates have a reality calibration.

---

*Released into the public domain under [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/). Reuse for your own protocol's roadmap if helpful.*

*The first peer is the hardest. The hundredth is the test. The thousandth is the success. — RFC-0010*
