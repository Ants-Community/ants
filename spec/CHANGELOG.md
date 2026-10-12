# ANTS RFC CHANGELOG

A record of substantive decisions and amendments to the RFCs. Per RFC-0001's
amendment process: clarity and typo PRs are not logged here; design-changing
amendments are. Each entry states what was, what is now, and why.

---

## RFC-0008 v0.9 + RFC-0005 v0.3 — Attestation presentation (§11.12), signed `issued_at`, v1 verify-and-tag admission · 2026-10-12

**Was:** RFC-0005 said a peer presents its attestation "on demand to other peers during handshake", but no document pinned the bytes. An implementer had no exchange, no envelope, no per-vendor blob layout and no rule tying the evidence to the connection. RFC-0005 §Attestation freshness window measured freshness from "the attestation's **signed** `issued_at` timestamp", yet no signed field carried it. The vendor-signed SNP report and TDX quote have no issuance timestamp, so `issued_at` existed only as unsigned presenter metadata. A stale attestation could be re-dated freely, and the 30-day window bounded nothing. Finally, the corpus did not say what a v1 peer should do at the handshake while attestation *generation* still waits on real confidential-compute hardware, and no peer can present evidence yet.

**Is now:** RFC-0008 §11.12 pins the presentation:
- **Exchange.** ATTEST_REQ / ATTEST_RESP (types 9/10) ride the §11.2 bidirectional-RPC envelope. The DHT message-type registry is generalized into one *bidi-RPC envelope* registry, so a receiver can demux by `type`. An ATTEST_RESP is either empty ("not attested") or `{0: vendor, 1: issued_at, 2: expires_at, 3: blob}`.
- **Blob layouts.** TDX uses the unmodified DCAP v4 quote with the PCK chain, verified to the pinned Intel SGX Root CA. SNP uses the report followed by the GHCB certificate table, verified to the per-family pinned ARK. Any ARK carried in the table is ignored.
- **Binding.** The 64-byte guest-chosen `report_data` is pinned as `peer-id ‖ LE64(issued_at) ‖ 0×24`, which is what makes `issued_at` the signed timestamp RFC-0005 already claimed. The key is never carried on the wire: the binding is checked against the peer-id authenticated by the TLS handshake on that connection.
- **Verification.** Five-step rule. Failure is a verdict, not an error, and there are three verdicts. *Attested* means all steps pass. *Not attested* covers an empty body, an unsupported vendor or stale evidence. *Rejected* covers a failed chain, signature or binding, or a malformed answer. RFC-0005 disconnects only on *rejected*. The presentation map is closed: an extension takes a new envelope type, not a new key, so old nodes never judge valid messages malformed.
- **Constants (§7).** `ATTESTATION_CLOCK_SKEW` (300 s, C), `ATTESTATION_MAX_BLOB_BYTES` (16384, P), `ATTESTATION_MAX_RESP_BYTES` (20480, P — both caps are pinned, since exceeding them is *rejected*).
- **§11.2 clarification.** An empty body is encoded by omitting key 2 (the envelope is `map(2)`), an explicit `{}` is non-canonical, and `txid` fits in 32 bits. These are the deployed DHT bytes since v0.6, and the tables had left them implicit. The attestation exchange follows the same rule.

RFC-0005 v0.3 cross-references §11.12 and adds the **v1 verify-and-tag posture**. An empty answer means the peer is admitted unattested. Verified evidence tags the connection attested. *Rejected* evidence means disconnect, while an unsupported vendor or stale evidence leaves the peer connected but unattested. Every attestation-requiring context (committee selection, trustee rotation, high-stakes bonds) still treats unattested as not attested. A network without attested peers is a trusted-seed testnet, and moving to hard-gate is a protocol-version event, not a config flag.

**Why:** The reference client has completed pure-software TDX and SNP verification and is wiring the handshake gate into its node daemon. That gate is interop-bearing, so its bytes belong in the spec before the code, decided in daylight.

The `issued_at` binding is the substantive call. Without it, RFC-0005's freshness window and the revocation-propagation bound built on it are unenforceable. The fix costs nothing at generation time, because the attested guest code chooses `report_data`. The design also leaves out a per-connection verifier nonce, which would force a fresh quote per connection, and keeps the replay model RFC-0005 already chose: window plus binding. Verify-and-tag is the founder's v1 decision. It lets the trusted-seed testnet run on the real gate code path without pretending to be Sybil-resistant.

## IMPLEMENTATION.md — TEE attestation (#3) un-deferred from v2.x into v1 · 2026-06-15

**Was:** The component #3 row, the critical-path section, the Phase-A
deliverable, and the parallel-work list all marked the TEE attestation harness
**"Targeted v2.x, not v1.0"** and **"deferred to v2.x per RFC-0005's
hardware-trust timeline (2030-2032)"**, with v1 shipping only a
`NOT_IMPLEMENTED` stub and verifiability resting on the other three legs of
RFC-0002 (re-execution, scheme (C), reputation). This was internally
inconsistent with RFC-0005, which assumes closed-vendor attestation is live
throughout the v1 era ("Until then, all attestation flows through the five
named closed-vendor families") and builds its entire Sybil-economics table and
2028-2030-window analysis on that assumption.

**Is now:** #3 is a **v1 deliverable**, and the roadmap records *why the
earlier deferral was wrong*. The TEE does two jobs and the deferral conflated
them. As the fourth *verifiability* leg (RFC-0002) it is genuinely optional —
the other three legs suffice, and that part of the reasoning stands. But as the
*identity root* it is not optional: RFC-0005's "one attested CPU, one peer
identity" and RFC-0004's "one CPU, one voice" validator pool both reduce to
*the attested population*, and with only the stub that population is every
freely-generated Ed25519 keypair — the L2 committee selection, the
per-attested-identity rate limits, and the Sybil-cost table are all unenforced.
The 2030-2032 horizon is RFC-0005's *open-hardware* migration
(Keystone/OpenTitan), not a justification for deferring closed-vendor
attestation. v1 staging: the two mature, publicly-auditable x86-server roots
**Intel TDX + AMD SEV-SNP** first (~2.4 of the component's 6 EM); ARM CCA, Apple
SE, Qualcomm follow as the device population broadens (v1.x). #3 stays
foundation-layer and parallelisable, so the `#7→#8→#13→#15` critical-path
length is unchanged — but it is now a prerequisite for integration testing and
the first testnet.

**Why:** A permissionless protocol whose identity root is a stub has no
hardware Sybil resistance at all — "one keypair, one voice" — a materially
different security posture from the one RFC-0005 advertises, and the corpus
stated both. The inconsistency is resolved toward RFC-0005 (the document that
owns identity) rather than toward the roadmap's deferral note. This reverses
the 2026-05-21 re-scoping that moved #3 to v2.x and dropped the v1 critical path
from 22 to 16 EM; the 16 EM chain is unaffected (TEE parallelises), but TEE is
no longer optional for a v1 that intends to ignite a trust-bearing network.
Quote *verification* is offline- and vector-testable on commodity hardware (the
security-critical path every peer runs); only quote *generation* and
end-to-end attestation need confidential-compute hardware, and are staged
accordingly. No RFC text changes — RFC-0005 already assumed this; only the
roadmap is brought into line.

---

## RFC-0008 v0.8 — ants-embed-v1 pinned to the BGE-M3 F32 GGUF · 2026-06-15

**Was:** §5 named the canonical embedding model by three values
(`embedding_model_id`, `_arch`, `_weights_hash`, `_tokenizer_hash`) but left the
two 32-byte hashes as `0x000…000` placeholders "to be set when the reference
client is published", and defined `_weights_hash = BLAKE3(canonical_safetensors_file)`.
The reference client's `cache/embedding` loader, however, consumes **GGUF**, not
safetensors — so the placeholders were unpinnable as written, and the artifact
the spec described was not the artifact a peer actually loads.

**Is now:** §5 pins the v1 values against the artifact the client loads. New
§5.1 records `embedding_model_weights_hash = 3055cf6c…5b0d26e6` (BLAKE3 of the
canonical **BGE-M3 F32 GGUF**, 2 273 655 072 bytes) and
`embedding_model_tokenizer_hash = a085385b…e1269cee` (BLAKE3 of the official
`BAAI/bge-m3` `tokenizer.json`, 17 098 108 bytes). The weights-hash definition
moves from "safetensors file" to "the canonical GGUF file the client loads";
provenance is documented (source = official `BAAI/bge-m3`; obtain =
`CompendiumLabs/bge-m3-gguf` `bge-m3-f32.gguf`, or reproduce via `llama.cpp`'s
`convert_hf_to_gguf.py --outtype f32`); precision is pinned to **F32** (any
quantized derivative is a different artifact, not `ants-embed-v1`); and the
embedding procedure is stated (tokenize → wrap `<s> … </s>` → BGE-M3 forward →
CLS-pool the `<s>` state → L2-normalize). A note separates the *pinned weights
file* (bit-exact, cross-platform) from *embedding-output* determinism (F32
reduction order — the open RFC-0009 problem).

**Why:** The reference client's BGE-M3 bring-up made the canonical embedding
real: the forward pass was validated against `llama.cpp` (cosine ≥ 0.999999) on
this exact F32 GGUF, and `ants_embed_init` now strictly verifies both hashes.
Pinning the model is a protocol-version-boundary act (RFC-0008 §9), and the
safetensors-vs-GGUF mismatch was a latent spec-vs-implementation divergence —
closed here toward the artifact the client actually loads, the same
direction-of-authority the v0.6/v0.7 upstreaming established. This is the last
spec-side piece of Component #11 (canonical embedding service).

## RFC-0008 v0.7 — Compact-summary encoding (§11.9) · 2026-06-12

**Was:** RFC-0004 §"Compact summaries for large peers" specified the *mechanism* — a signed list of `(time_bucket, total_decayed_value)` pairs, committed via the same Merkle root, auditable bucket-by-bucket, false buckets being equivocation — but no wire encoding existed: §11.9 covered the receipt body, the bag commitment, and the `A ≥ b` openings, and stopped there. Two questions the prose left under-determined for an implementer: what "committed via the same Merkle root" means operationally (a summary leaf inside the tree would make the root self-referential), and *at what time* the bucket totals' decay is evaluated — load-bearing because the pinned q32 decay is a repeated multiplication and not associative, so totals re-decayed to a later time would drift from a fresh recompute by rounding and never compare bit-exactly.

**Is now:** §11.9 gains the compact-summary encoding: a 5-pair signed body (peer_id, bag_root, eval_time, bucket_width, buckets) re-keyed as a 6-pair full object with the peer's single Ed25519 signature — the summary is the peer's own statement about its own bag. Canonical form: strictly ascending bucket indices, zero-total buckets omitted, at most `SUMMARY_MAX_BUCKETS`. The two under-determined points are pinned: "committed via the same Merkle root" = the signed body *includes* `bag_root` (key 2), and totals are decayed **at the summary's own `eval_time`** with audits recomputing at that same instant, bit-exactly — re-decay is explicitly ruled out, fresher numbers mean a fresher summary. The audit rule is stated (every opening verifies as for `A ≥ b`, additionally falls in the audited bucket and predates `eval_time`; revealed contributions must reach the stated total, exceeding it being the safe understatement direction). §7 adds `SUMMARY_MAX_BUCKETS` (256, pinned) and `SUMMARY_BUCKET_WIDTH` (86400 s, DRAFT calibratable default — RFC-0004 already made granularity a calibration question); §8's vector list covers summaries.

**Why:** This was the last RFC-0004 selective-disclosure surface implemented by the reference client (ants-client `reputation/identity`: build, codec, verify, per-bucket audit, shipped with this amendment) and the last piece of Component #9's lib surface — the same freeze-it-now reasoning as v0.6: format decisions an independent implementation would otherwise have to reverse-engineer from C comments belong in the spec, decided in daylight. The eval-time pin is the substantive call: it is what makes a compact summary *auditable at all* under the bit-exact-or-reject discipline, and it falls out of the same non-associativity fact the reputation spine already documents.

---

## RFC-0008 v0.6 — Protocol message encodings upstreamed from the reference client · 2026-06-12

**Was:** RFC-0008 pinned the primitives (canonical CBOR profile, BLAKE3, Ed25519/BLS, ECVRF, drand failover, constants) but no concrete message format: every object the reference client actually exchanges — DHT RPCs, gossip frames, L1 fault-proof envelopes, L2 epoch summaries and blocks, the commit-at-send object, the answer envelope, receipt bodies, the receipt-bag commitment, bond admissions — lived in `ants-client` module headers under an explicit "WIRE FORMAT IS DRAFT, pending formalisation in RFC-0008" marker. Two spec-vs-implementation divergences had quietly accumulated: §4.1 reserved `derive_key` contexts for the Merkle trees (`ants-v1-fault-merkle`, `ants-v1-logit-trace-*`) and the beacon challenge derivations (`ants-v1-beacon-*`) that the implementation never bound — it converged instead on prefix-byte Merkle domain separation (RFC 6962 style) and a counter-mode BLAKE3 keystream with 3-byte tags. Four contexts the implementation or other RFCs *do* use (`ants-v1-vrf-seed`, `-genesis`, `-bond-tiebreak`, `-receipt-bag-root`) were missing from the table, and §10 still listed the drand round mapping as unresolved when §4.3 had pinned the rule and the implementation had pinned the value.

**Is now:** New **§11 "Protocol message encodings (reference)"** specifies, byte-level: the transport identity binding (QUIC v1, ALPN `"ants"`, RFC 7250 raw-pubkey TLS 1.3 with the exact 44-byte Ed25519 SPKI, multiaddrs, per-subsystem stream discipline); the DHT RPC envelope and all eight message bodies; the three gossip frames plus the length-prefixed persistent-stream framing and the late-joiner snapshot; the canonical prefix-byte Merkle construction with its three instantiations in one table; the L1 fault-proof envelope, statement body, and both assigned evidence schemas; the L2 epoch summary, finding, and block layouts with the attestation/finality/proposal rules; chained-drand round verification; the commit object, the challenge keystream, and the answer envelope; receipt bodies and the receipt-bag selective-disclosure commitment; and the bond-admission statement with its tie-break key — closing with the append-only wire registries (fault classes, statement domains, severities, frame/message types, tiers). §4.1 gains the four missing contexts plus an explicit binding-status note naming which reserved strings the implemented constructions supersede; §4.2 pins `drand_round_value` as the round's 32-byte randomness; §2.2's two-hash-universes rule now covers drand; §7 adds the DHT/gossip/receipt wire constants; §8 records that the test-vector repository exists and which packs it ships; §10 swaps the resolved drand-mapping item for the genuinely open ones (derive_key migration, the weight-blob's RFC-0009 appendix). "How to contribute" renumbers to §12 (nothing referenced §11). RFC-0009 §5.1(c) is updated to cite the implemented leaf rule instead of the superseded contexts.

**Why:** Fourteen of fifteen reference-client components are feature-complete and the #8 protocol logic closed this morning — the moment the accumulated DRAFT surface is largest and freshest, and the last cheap moment to freeze it: every format here was marked "MUST be upstreamed before two independent implementations interoperate," and the spec-vs-implementation divergences would otherwise harden silently into two incompatible readings of the same document. The direction of authority is now explicit and inverted: the spec owns the bytes, the reference client conforms. Reconciliation went *toward* the shipped, tested constructions (prefix-byte Merkle, challenge keystream) rather than re-pinning crypto to match aspirational table rows — the honest order, recorded as such, with the migration question left explicitly open in §10 rather than decided by silence.

---

## RFC-0004 v0.7 — Committee fabrication made attributable: the invalid-transition fault · 2026-06-12

**Was:** §"Why the committee cannot fabricate" argued fabrication is "trivially detectable by any peer running VERIFY independently" — detectable, but not *attributable*: the `invalid-transition` fault class had been reserved in the Layer-1 enumeration since Component #7 PR1 with no specified semantics, pending the L2 chain. The recorded design fork: (i) self-contained block evidence — but checking block *finality* needs the committee's public keys, which need the attested population and the beacon, i.e. Layer-2 context a context-free VERIFY cannot reach; vs (ii) an L2-side fault class outside the G-Set contract — keeping Layer 1 pure but losing the one-honest-path propagation guarantee for exactly the fault that polices Layer 2.

**Is now:** RFC-0004 v0.7 §"Fabrication is attributable" specifies the class as a **signer-attributable fabrication proof**, a third shape that dissolves the fork: the fault is *"the subject signed the hash of a block whose epoch summary commits, in its confirmed_proofs root, bytes that are not a valid fault proof"* — requiring neither membership derivation nor finality, because the signature is the culpable act, exactly as an equivocation signature is. Evidence is self-contained (canonical block + signature over its hash + cited bytes + Merkle inclusion path); VERIFY stays pure, deterministic, context-free. Honest members are structurally safe: their candidate blocks are recomputed from their own VERIFY-gated G-Sets (recompute-and-compare is proposal validation). Two bounds are stated honestly — no nesting (a citation cannot itself be invalid-transition; committing an invalid invalid-transition proof stays detectable-but-not-attributable, like omission) and cross-version permanence (decode failure under the pinned RFC-0008 §1.1 profile and deployed-class MALFORMED verdicts are permanent → slashable; an unknown-class citation is a typed "don't know" → never a slash, so no build ever admits a proof whose citation another build deems valid).

**Why:** This was the last reserved Layer-1 fault class whose dependencies (the Component #8 block/Merkle codecs) now exist, and resolving it turns the chain's load-bearing property — "the committee cannot fabricate" — from an argument about detectability into an enforced economic fact: fabrication costs the fabricators their identities the moment one honest peer sees the block, propagating on the same one-honest-path guarantee as every other fault proof, with no new propagation channel and no weakening of the Layer-1 contract. Reference implementation: ants-client `reputation/crdt` (VERIFY + encoder), cross-checked bit-for-bit against the `reputation/chain` Merkle and block codecs.

---

## RFC-0005 v0.2 — Anti-eclipse peer sampling specified · 2026-06-02

**Was:** Anti-eclipse peer selection was named as RFC-0005's responsibility by RFC-0004 (§"What we have not figured out yet") but left unspecified: the gossip overlay's view (Component #6) was caller-managed with no defined source, so the one-honest-path property it underwrites had no mechanism behind it.

**Is now:** RFC-0005 gains §"Anti-eclipse peer sampling", which fixes the *mechanism* on two axes of choice. **Placement:** the Sybil-resistant sample is a DHT (Component #5) primitive — keeping the gossip engine substrate-agnostic and the security-critical selection testable in one place — rather than embedded in the gossip engine (layer coupling, untestable without a live DHT) or scattered across per-deployment integration glue (untested). **Selection:** diversity from three independent axes — a bucket-diverse routing-table draw, random-target probes, and view rotation — with **attestation-weighted selection named as the strong fourth axis but deferred** until the attestation flow is deployed. The recursion (anti-eclipse ⇄ Sybil-resistance) is stated and bounded by the one-path-not-majority asymmetry. RFC-0004's open-item bullet is updated to point here (a cross-reference, no RFC-0004 version change).

**Why:** Component #6 (gossip) had shipped every piece that needed no cross-layer decision (engine, transport binding, lazy-pull, persistent channel, T_prop instrumentation, local reject accountability); anti-eclipse was the remaining architectural fork — *where* sampling lives and *what* it samples on. Deciding it unblocks the reference client's view wiring and stops RFC-0004 from leaning on a wholly-unspecified assumption.

**Explicitly NOT closed:** a *provable* eclipse-probability bound. Axes 1–3 are defence-in-depth heuristics; the rigorous guarantee waits on attestation-weighted selection (axis 4) plus a formal model. RFC-0004's one-honest-path assumption is narrowed and made costly, **not discharged** — and must not be reported as resolved.

---

## IMPLEMENTATION.md — P2P transport: picoquic, not libp2p · 2026-05-21

**Was:** Component #4 (P2P transport) listed two paths — "3 EM via libp2p, 6 EM custom" — with the recommendation to use libp2p for time-to-implementation, deferring a custom stack to v2.0. The libraries table left the choice as an open question between custom C, libp2p-c (C++ binding), or an external daemon via IPC.

**Is now:** **Stack pinned to vendored [`picoquic`](https://github.com/private-octopus/picoquic) (IETF QUIC reference, pure C99, BSD-3) + ANTS-native application protocol on top.** The component descriptor is rewritten to reflect this. EM estimate revised from "3 or 6" to 5: ~3 EM to wrap picoquic into the ants-native transport API, ~2 EM for the application protocol primitives (peer routing, stream framing) that previously would have come from libp2p.

**Why:** During foundation/network architectural review, the libp2p-c path turned out to be impractical for a C99-pure reference client. The canonical libp2p implementations are Go (go-libp2p) and Rust (rust-libp2p); the C++ binding (cpp-libp2p) is half-maintained and pulls in a C++ runtime that violates the project's day-zero portability commitment. The external-daemon-via-IPC option carries the same runtime dependency wrapped in process boundaries. By contrast, picoquic is the IETF QUIC reference implementation in pure C, ~50k LOC, BSD-3, snapshot-vendorable in the same shape as `deps/blst`/`deps/blake3`/`deps/ed25519`. We get the well-specified parts (TLS 1.3 handshake, stream multiplexing, congestion, NAT-friendly UDP transport) from picoquic, and we write the ANTS-specific application protocol (peer identity binding via Ed25519+RFC 7250 raw-public-key, DHT query framing, gossip discipline) in our own C99. This is closer in spirit to "we control our protocol layer" than wrapping libp2p's opinionated peer/routing/pubsub stack.

The amendment is documented in IMPLEMENTATION.md component #4 row and the libraries table; the rationale lives in the row's prose.

---

## IMPLEMENTATION.md — TEE harness deferred to v2.x · 2026-05-21

**Was:** Component #3 (TEE attestation harness) was on the v1.0 critical
path, listed as "the single longest foundation component" (6 EM), with a
[CLAIM] issue (#6) treating it as a near-term nomination target.

**Is now:** Targeted v2.x explicitly. Component descriptor #3 marked as
deferred with citation to RFC-0005's hardware-trust timeline (current
silicon-generation vulnerability windows close around 2030-2032, when
TEE attestation as a participation path becomes operationally meaningful).
v1.0 ships with a `NOT_IMPLEMENTED` stub for the TEE API surface so
upstream components can integrate against it; verifiability rests on the
other three legs of RFC-0002 (re-execution, scheme (C) probabilistic,
reputation). Critical-path chain shortened from 22 to 16 engineer-months.

**Why:** The 2026 nomination window for the TEE [CLAIM] issue was creating
an artificial pressure inconsistent with the 2030-2032 timeline already
adopted in RFC-0005 Round 1. Anchoring TEE explicitly to v2.x removes that
pressure, lets the community nominate a primary on a realistic schedule
(years, not weeks), and lets v1.0 ship with full RFC-0002 verifiability
even without TEE-as-a-fourth-leg.

## RFC-0001 — Barter · v0.1 → v0.2 · 2026-05-19

Two amendments, both forced by discoveries in the later RFCs (RFC-0002 through
RFC-0004) once the full corpus closed. RFC-0001 was **not** rewritten; only these
two points changed.

### 1 · Ledger portability: open question → reasoned trade

**Was (v0.1):** an open question — *"each peer's view is private to it, and that's
deliberate… portable, attested credential? We lean toward 'OK for v0.1, revisit in
v0.3.'"*

**Now (v0.2):** reclassified from a deferred convenience to a **security
parameter**. RFC-0004's fork-recovery showed the credible-fork-threat — the
architecture's fixed point — fails if a fork resets earned standing, so portability
is load-bearing, not optional. The portable credential is RFC-0003's
selective-disclosure threshold proof (commitment to the receipt bag + a
zero-knowledge proof that tenure ≥ floor). It is sound because reputation is
asymmetric: positive evidence is holder-presented and selective disclosure can only
*under*-claim; negative evidence (slashes) is not in the holder's bag — it is
RFC-0003's non-suppressible global set — so faults cannot be hidden. Irreducible
cost: there is no zero-leakage portable reputation; the one bit "established vs
newcomer" is unavoidable, and it is exactly the signal cold start and identity
functionally require. Full portability and full ledger-privacy are not jointly
satisfiable; minimised-disclosure portability is the chosen trade.

### 2 · S_NCS: assumed parameter → derived quantity

**Was (v0.1):** undefined. RFC-0002's deterrence threshold needed "the value of the
standing a slash destroys" and flagged it as its weakest, endogenous input.

**Now (v0.2):** derived. A slash's durable cost is κ-rate-capped tenure re-accrual:
`S_NCS ≈ w · TENURE_FLOOR / κ` (a lower bound; conservative — the true deterrent is
stronger). `w`, the establishment premium, is the only residual and is measurable
on the testnet, not assumed. Consequence: `S_NCS ∝ 1/κ` ⇒ RFC-0002's threshold
`T ∝ √κ` ⇒ a smaller κ strengthens **both** Sybil resistance (RFC-0004) and
verification viability (RFC-0002). The two hard layers align in κ rather than
trading against each other: **κ is the single master security knob of the
architecture.** This closes the endogeneity RFC-0002 flagged; it does not close the
patient single-decisive-act residual.

---

## Architectural pivot · corpus v0.1 → v0.2 · 2026-05-20

A corpus-level restructure with several pieces. Recording it here because it is
exactly the kind of design-changing amendment RFC-0001's own process prescribes a
CHANGELOG entry for, and the pivot commit landed without one — a process gap this
entry closes.

The pivot has six pieces. The first three are structural — the cache, the economic
split, the renumbering. The last three are substantive design choices in
individual RFCs. The animating principle, applied to all six: where v0.1 had
derivations and proofs, those survive as the technical mechanism beneath a new
consumer-facing surface. No prior result is discarded silently. Where a prior
result is rejected, it is rejected by argument, here.

### 1 · RFC-0002 Semantic Cache · new layer

**Was:** no cache layer in the corpus.

**Now:** RFC-0002 (Semantic Cache) introduced as a foundational layer — a
distributed content-addressable store keyed by prompt embedding. Every honest
answer the network has given lives there, signed by its producer, retrievable by
similarity.

**Why:** The deepest economic critique of the v0.1 corpus identified a structural
pathology in pure barter: specialists whose expertise is rare and whose demands
are modest accumulate unspendable credit, because asymmetric demand cannot be
settled bilaterally. The cache resolves this directly: a high-quality answer
becomes durable yield rather than a one-time payment, paid in fractional NCS to
the original producer on every subsequent retrieval. It is the correct response to
the real problem, and it does so without introducing money, tokens, or central
coordination.

### 2 · RFC-0001 split into Community Economics + RFC-0006 Payment Terms · v0.2 → v0.3

**Was:** a single RFC-0001 (Barter) implying barter was the protocol's only
economy.

**Now:** RFC-0001 (Community Layer Economics, v0.3) specifies barter for the
community population. RFC-0006 (Payment Terms) specifies the metadata field by
which any peer declares its economic terms — barter, fiat, subscription, gift, or
custom — and the rules for coexistence. Multiple economies share the transport
layer, the cache, and the identity layer; they are isolated economically and
interoperable everywhere else.

**Why:** Another v0.1 critique flagged that "no money at the protocol layer"
combined with "cloud providers can sell access on top" silently recreates the
centralization the manifesto refuses, one layer up. The HTTP analogy applies: HTTP
does not know what payment is, and that neutrality is why it carries every
economy. The protocol declares no economy; populations declare their own; the
foundation captures none of them.

### 3 · RFC renumbering

**Was:** RFCs 0001–0004 covered Barter / Verification / Reputation / Identity.

**Now:** RFCs 0001–0006 cover Community Economics / Semantic Cache / Verification
/ Reputation / Identity / Payment Terms. RFC-0007 (Post-v1.0 Governance) is
reserved.

**Why:** the cache becomes architectural peer to the economy, not a sub-component.
Payment Terms peer to the economic layer rather than nested in it. Numerical
contiguity is a minor cost worth paying for structural clarity. The old files
under the prior numbering have been kept temporarily for diff and are scheduled
for removal in a follow-up.

### 4 · RFC-0003 Verification · v0.1(early) → v0.2 · marriage of tier framing and scheme (C)

**Was (v0.1 early):** a three-tier consumer-facing menu (attestation only /
cross-check / triangulation) with key parameters declared as needing calibration —
sampling rate 3%, cross-check cosine threshold 0.85, and so on — and no
anti-grinding analysis of the challenge mechanism.

**Was (in prior v0.1 derivation cycles):** a single-mechanism design — scheme (C),
commit-at-send · prove-if-challenged · lose-everything-if-caught — with: a
bounded-discrepancy aggregate sequential test (an anytime-valid betting e-process,
controlled by Ville's inequality with no independence or variance assumption);
the closed-form economic viability threshold `T = √[2τG·log(1/α) / (ρ·L·S_NCS)]`
relating effect size to the deterrence economics of RFC-0001; a binding
cross-layer constraint linking the slash propagation window to the verifier
rotation cadence; the canonical-numerics requirement on the audited path as the
pivotal design lever; and a pre-registered two-gate testnet experiment (the
"safety null first, viability second" protocol called b2 internally).

**Now (v0.2):** both, integrated. The three-tier menu is the outer, consumer-facing
structure — it stays, because it is the right way to expose a cost/assurance
tradeoff to the requester. The unified primitive beneath all three tiers is
commit-at-send: Merkle binding of the full logit trace, beacon-bound challenge
selection (anti-grinding), beacon-derived auditor assignment (anti-collusion).
Tier 1's "3% sampling" becomes a default *example* with its deriving inequality
made transparent (the sampling rate `p` is bounded below by the deterrence
condition and above by the budget, both derived from RFC-0001 economics). Tier 2's
"cross-check" is scheme (C) properly stated, with the betting e-process replacing
the per-position cosine threshold of the v0.1(early) draft. Tier 3's triangulation
has each of the N committee peers publish independent commit-at-send commitments,
so the anti-grinding guarantee is per-peer and the convergence test sits on top of
it. Canonical numerics is a hard requirement for any tier ≥ 2. The b2 experiment
opens the document as the empirical decision rule.

**Why:** the v0.1(early) draft introduced a much better user-facing surface than
any prior draft had — the explicit tier-by-tier menu is genuinely good design, and
it covers cases (Tier 3 triangulation, human feedback for subjective domains) that
the earlier single-mechanism design did not. But its parameters were guesses, and
its cross-check rule (per-position cosine) had been proven inadequate against the
make-or-break adversary class (FP8-degradation fraud, where honest hardware noise
and dishonest quantization overlap per-position and are separable only in
aggregate). The earlier work derived the rigorous mechanism that fixes the
parameters and the cross-check; it was published and survived independent review.
Discarding it and replacing the parameters with guesses would cede ground to
expert critique that the project had already earned. Integrating keeps both: the
explicable surface, the defensible mechanism. No funeral. A marriage.

### 5 · RFC-0004 Reputation · v0.1(early) → v0.2 · two layers, witness over CRDT

**Was (v0.1 early):** a single mechanism — a small slow blockchain with novel
"Proof-of-Unique-Hardware" (PoUH) consensus as the primary structure. Slashing
arises from finalized chain blocks. Hardware attestation provides Sybil
resistance; VRF selects committees of K=64 from the attested peer population;
2/3 committee signatures finalize a block.

**Was (in prior v0.1 derivation cycles):** a consensus-free design — a grow-only
set (G-Set CRDT) of self-authenticating monotone fault proofs. `VERIFY(π)` is a
deterministic context-free pure function: no two honest peers can disagree on
the validity of an individual proof. Slashing is immediate, locally, per peer who
receives a proof. The only convergence required is set convergence (gossip
propagation, `O(c·Δ·log N)` under anti-eclipse) — not consensus. Critically, a
self-authenticating monotone fact needs *one honest path*, not an honest majority.
The trust-closed argument of the corpus terminates here in a bounded-time social
coordination assumption, not in a committee.

**Now (v0.2):** two architecturally distinct layers, one architecture, with a
clear division of labor. **Layer 1** is the consensus-free G-Set CRDT of fault
proofs. Anyone who receives a proof slashes locally and immediately, no consensus
needed. The chain has no role in individual slashing. **Layer 2** is the PoUH
chain, redirected to a different job: its committees, per epoch, attest to the
state of the CRDT at the snapshot time — confirming which fault proofs have
propagated, and which peers have crossed rate- or pattern-based thresholds ("6
medium-severity events in 30 days"). The chain is *witness*, not *judge*. It
cannot fabricate slashes (the proofs remain self-authenticating; VERIFY rejects
forgeries regardless of what any committee signs); it can only fail to witness
true ones, in which case the next epoch's committee picks up the backlog.

**Why:** the v0.1(early) draft provided something the consensus-free design did
not: a clean mechanism for global ordering of events, which makes rate-based and
pattern-based slashing rules straightforward to specify. Pure CRDTs do not order;
ordering is what a chain is for. But the v0.1(early) draft made the chain
*load-bearing for individual slash propagation*, which reintroduces the
honest-majority assumption and weakens the manifesto reconciliation (Thesis 13's
"no central authority" lives in tension with Thesis 10's "consensus layer
assigns each attested machine one voice"). The two-layer design keeps what each
mechanism does best: the CRDT for fast consensus-free slashing under the weakest
possible assumption; the chain for ordered, finalized epoch-summaries of state
that has already been established by the CRDT.

One consequence is graceful degradation: a compromised PoUH committee for one
epoch cannot create false slashes (the proofs are still self-authenticating); it
can only delay witnessing. A pure-chain design would have suffered a more serious
failure under the same compromise. A second consequence is bootstrap softening:
the CRDT works from day one with one honest path; the PoUH chain matures as the
attested population grows past the size at which committees of 64 become
statistically defensible. The genesis bootstrap is no longer a forced choice
between "we have a chain" and "we have nothing."

On novelty: "PoUH consensus" presented alone does not engage with prior art —
Oasis Network's confidential validators, Phala's TEE-attested workers, the broader
family of VRF-selected committee BFT mechanisms. The honest novelty claim is
narrower and stronger: it is the *combination* of PoUH-attested committees as an
ordered-witness layer above a consensus-free CRDT of self-authenticating proofs.
That combination, as far as the present draft authors know, is new, and it is
substantially more defensible under expert critique than the
consensus-mechanism-alone framing.

### 6 · Manifesto edits

**Was:** sixteen numbered theses, with "we refuse to issue a token" as an absolute
and "we build by barter" implying barter was the only economy.

**Now:** seventeen theses. Thesis 7: "we build *a community* by barter" — the
community's economy, not the protocol's only one. New Thesis 9: "we build memory
into the network" — explicit endorsement of the semantic cache. Thesis 10: "the
protocol's consensus layer assigns each attested machine exactly one voice" —
explicit endorsement of PoUH, scoped to the witness layer. Thesis 15: "we refuse
to issue a token *at the protocol layer*" — allowing economies on top to use
whatever instruments they declare. New Thesis 17: "we refuse to extract rent" —
the foundation does not capture the network it built.

**Why:** the original wording, taken literally, would have made the cache and the
economic pluralism inconsistent with the manifesto itself. The edits move from
absolute refusals to layered ones — refusing at the protocol layer what we cannot
allow there, permitting on top what we cannot, and should not, forbid.

A residual tension is acknowledged and not erased: Thesis 10's consensus layer is
in literal tension with Thesis 13's refusal of central authority. RFC-0004 v0.2's
two-layer architecture reconciles them in practice — the PoUH layer is rotating,
non-load-bearing for individual slashes, and gracefully degradable — but the
reader is invited to test whether the reconciliation is sound.

---

## RFC-0004 · v0.2 → v0.3 · 2026-05-20 · A-as-bond for high-stakes acts

A focused amendment closing — partially, and honestly — a residual that the
v0.2 "strategically inert" argument left implicit.

### Was (v0.2)

The §Tenure section presented a trilemma against farmed tenure: a Sybil with
mature `T` faces three branches — never used (zero influence), used honestly
(it *is* the work the network wanted), used to collude (attributable fault
zeroes the tenure). The argument was claimed complete ("no fourth branch").

It was not complete. It implicitly assumed the one-shot payoff of a collusive
act is less than the slash value `S_NCS`. For high-stakes acts — Tier 3
verification on a multi-million-dollar query, a fork-recovery vote, a
perennial high-value cache write — the single-act payoff can exceed `S_NCS`.
A defector with mature tenure could execute such an act once, accept the
slash, and come out ahead. The branch existed; it was just not named.

A related clarification on the framing in earlier draft cycles: the
"patient adversary farming 10,000 Sybils for years" scenario was imprecise.
Under RFC-0005's PoUH, each Sybil requires real attested hardware — 10,000
attested identities cost the floor RFC-0005 tabulates (~$7M/year in
cloud-TEE rental, more for owned hardware). PoUH already constrains the
mass-Sybil patient farming variant; what it does not constrain is *one
legitimately-attested identity that defects once decisively*. That is the
residual that this amendment addresses.

### Now (v0.3)

A new mechanism: **A-as-bond for high-stakes acts**. Acts whose plausible
one-shot payoff could exceed `S_NCS` require the actor to lock a quantity of
`A` (active reputation) as a temporary bond, slashed if the act is later
determined malicious. The bond is released after a dispute window if no fault
proof emerges.

The mechanism exploits the (A, T, κ) split already in the spine. `A` decays
fast (δ_A, minute-scale) and **cannot be accumulated in advance**. Tenure can
be farmed slowly under κ; active reputation cannot. A defector with mature
`T` but neglected `A` therefore cannot bond enough for a high-stakes act.
The only path to a bond large enough is sustained recent honest participation
— which is the behavior the network wanted. The "patient farm + one-shot
cash-in" path is closed.

RFC-0004 v0.3 enumerates the act classes that require bonds (Tier 3
verification committee membership, L2 PoUH committee role, perennial
high-value cache writes, fork-recovery votes, cross-economy settlement
intermediation) and specifies the bond formula per class.

The trilemma becomes a quadrilemma — explicit fourth branch added: *used to
collude at high stakes → requires an A-bond ≥ the payoff; the bond can only
be assembled by sustained recent honest participation; therefore the cost of
attempting the act is at least the cost of being honest first*. The
inversion-by-amortisation argument that runs through the corpus reapplies at
this finer grain.

### Why

The earlier framing implied that the patient-adversary problem was mass
quantity. It is not. PoUH (RFC-0005) bounds the mass case at hardware-cost
floors that meaningfully deter all but state-actor budgets. The real
residual was always single-identity defection on payoffs higher than the
standard slash value. A-as-bond is the structural mechanism that closes this
because — by spine construction — `A` is the resource that cannot be bought
cheaply in advance. The defence reuses the existing (A, T, κ) primitive
rather than adding a new one, which is the small-surface-area discipline the
manifesto asks for.

What remains, acknowledged: state-actor-scale adversaries with both the
budget for genuine sustained participation and a specific high-value target
whose payoff exceeds bondable `A`. This is the irreducible residual — the
same shape as every "trust-closed" claim in the corpus. The amendment does
not claim to close it. It claims to narrow it.

---

## RFC-0009 v0.1 → v0.2 + RFC-0002 v0.1 → v0.2 · 2026-05-20 · Y_canon caching + embedding governance

Joint amendment addressing three open BLOCK items from the post-0008/0009
criticality re-check: B-NEW-1 (producer compute economics), B-NEW-2 (cache
content Y_canon vs Y_fast), and B5 (embedding model upgrade governance from
the original pre-implementation review).

### B-NEW-1 + B-NEW-2 — RFC-0009 v0.2 + RFC-0002 v0.2

**Was:** RFC-0009 v0.1 said "the producer serves whatever it prefers … and
additionally commits to the canonical output." This was ambiguous about
what the user receives and what gets cached. Implicit consequence:
producers paying 2× compute on every served query (not the +6%
network-overhead figure RFC-0009 v0.1 cited at the producer level), and a
silent question of whether the cache stores `Y_fast` (user-visible but
unverifiable) or `Y_canon` (verifiable but divergent from what the user
got).

**Now:** the canonical recipe is the unified output. What the producer
commits to, what the cache stores, what the user receives — all `Y_canon`.
The producer's internal serving path is an implementation choice: if it
produces `Y_canon`-equivalent outputs natively, the 2× tax is zero; if
not, the tax applies but is the producer's economic decision.

The cache property that every retrieval returns bit-identical bytes for
the same entry is a *strength*, not a bug. It makes Tier 2 audits of cache
hits trivial — under the integer-canonical recipe, σ is essentially zero
and any discrepancy is a slash. And it forces network-wide convergence
on the canonical recipe as the production serving path, which is the
intended outcome of the whole verifiable-inference layer.

**Why:** removes the silent ambiguity about user/cache divergence; makes
the producer economy explicit and binary (zero tax or full tax, no
hidden middle); gives Tier 2 audits a clean property to rely on. The
v0.1 framing was technically defensible but operationally underspecified.

### B5 — RFC-0002 v0.2 §Governance of the canonical embedding model

**Was:** RFC-0002 v0.1 named the canonical embedding model as "the
single most consequential governance decision the cache layer makes"
but did not specify the process. The flagship governance object of the
protocol's only true coordination point was left as a hand-wave.

**Now:** a new §Governance of the canonical embedding model specifies
the proposal/review/decision/transition flow:

- **Proposal**: minimum-content requirements (model hash per RFC-0008,
  perplexity comparison on a multi-language multi-domain benchmark,
  vulnerability analysis, licensing analysis, re-embedding compute
  estimate).
- **Review**: minimum eight-week public comment period (vs the standard
  one-to-three for ordinary amendments), written assessments from
  verification/identity/cache subsystem maintainers.
- **Decision**: BDFL during v0.x; two-thirds TSC majority post-v1.0
  (higher than the simple-majority for routine matters).
- **Transition**: six-to-twelve month coordinated dual-running, with
  background re-embedding incentivised by NCS.
- **Right to fork** throughout — the protocol's permissionless property
  applied to the most consequential governance act.

**Why:** the canonical embedding model is a single point of
coordination that the protocol genuinely depends on. The governance
process does not dissolve that dependency — it bounds the *abuse* of
it. The amendment makes the process explicit and public, so it cannot
be smuggled into by anyone, BDFL included. This is also the first time
the corpus distinguishes between "ordinary amendment" and "elevated
amendment" governance — a template for any future protocol-coordination
decisions of similar scope.

### What this does not close

Four BLOCK items from the pre-implementation criticality review remain
open after this amendment: B2 (bond accounting model), B4 (bootstrap
sequence), B6 (fork-recovery bond circularity), and the cross-RFC
nits/H-NEW items from the post-0008/0009 check. They are addressed in
Amendments B, C, and D following this one.

---

## RFC-0008 + RFC-0009 · added · 2026-05-20 · Technical references for implementation

Two new RFCs added together, in service of the same goal: close the
pre-implementation criticality gaps that would otherwise force the
reference-client engineers to guess on dozens of byte-level choices and one
genuinely hard numerical-recipe choice.

These are not design RFCs in the same sense as 0001–0006. They are
prescriptive engineering references — the boring glue that the design RFCs
presupposed but never wrote down. We add them now, before code begins, so
that "the spec" and "the implementation" can speak the same language.

### What was missing

A pre-implementation cold-read of the v0.3 corpus identified six BLOCK-level
gaps that would have stalled coding within weeks: an undefined canonical
numerical recipe (RFC-0003 named it as a leverage, did not specify it); an
unspecified TEE attestation wrapper format across 6 vendor families; an
absent bootstrap sequence; an unspecified embedding-model versioning
mechanism; an under-specified bond accounting model; and the fork-recovery
bond circularity. Several GAP-level issues were equally blocking in
aggregate: undefined PRF, undefined hash function, undefined signature
scheme, undefined wire encoding, undefined NCS representation, undefined
test-vector format.

The design RFCs left these underspecified for honest reasons — they are not
*design* questions, they are *engineering* questions, and writing a
prescriptive engineering reference inside a design RFC dilutes both. But
without them, the implementation cannot start.

### RFC-0008 — Wire Formats, Crypto Primitives, Reference Constants (Reference v0.1)

A living-standard technical reference that pins, in one place:

- **CBOR** ([RFC 8949](https://www.rfc-editor.org/rfc/rfc8949)) with
  deterministic encoding for protocol objects; **JSON** for human-facing.
- **BLAKE3** for protocol-internal hashing (Merkle trees, content
  addressing); **SHA-256/384** preserved for TEE-attestation compatibility.
- **Ed25519** for peer signatures; **ECDSA-P256** preserved for TEE;
  **BLS12-381** for L2 PoUH committee signature aggregation.
- **BLAKE3.derive_key** as the PRF for beacon-bound selection, with a
  reserved domain-separation table.
- **ECVRF-EDWARDS25519-SHA512-TAI** ([RFC 9381](https://www.rfc-editor.org/rfc/rfc9381))
  for VRF committee selection; **drand** (League of Entropy) as external
  entropy beacon.
- **u64 micro-NCS** (μNCS, 10⁻⁶ NCS) for all ledger arithmetic; floats
  forbidden in protocol-level objects.
- **Embedding model pinning mechanism** — BGE-M3 weights+tokenizer hashed
  with BLAKE3, exact checkpoint hash filled when the reference client
  ships.
- **A consolidated reference-constants table** of every parameter scattered
  across 0001–0006, with explicit Pinned vs Calibratable markers.
- **Test-vector framework** — a sibling repo, structure declared here,
  populated by the reference client.

Status is **Reference (living standard)** rather than **Draft** because the
document's relationship to argument is different — once a primitive is
pinned, it changes only via a coordinated breaking event, not via the
"written to be argued with" cadence of the design RFCs.

### RFC-0009 — Canonical Numerics for Verifiable Inference (Draft v0.1 early)

The hardest deliverable in the corpus, made explicit. The recipe RFC-0003
named as a leverage but did not specify, now specified — partially, and we
say so plainly.

**Default recipe**: INT8 weight quantization (GPTQ per-channel symmetric,
group 128) with integer-domain activations and pinned reduction order on the
audited path. Integer arithmetic is associative; two honest implementations
on different hardware produce bit-identical outputs. The `σ` honest-noise
floor under this recipe is essentially zero, which makes the e-process test
of RFC-0003 §Tier 2 trivial for any non-zero quantization fraud.

**Fallback**: FP16 with constrained kernel selection and deterministic
reduction order for models where INT8 quantization is not viable (some QAT
models, exotic architectures). σ is small (10⁻⁵ to 10⁻⁴) but not zero;
the e-process still works, with worse constants.

**Verifiability tax** quantified: the audited path is 2–4× slower than the
producer's fast-path serving. At p=3% sampling, this is +6% network compute
overhead. Borne by producers, not consumers.

**Reference kernel library** specified as `ants-canonical-kernels` (Rust,
hand-tuned SIMD per platform). Skeleton given; implementation is real work
— estimated 6–12 months for the CPU canonical path alone, with GPU
acceleration following per-platform. Until it exists, "canonical numerics"
is words; we say so in the RFC.

Marked **Draft v0.1 (early)** because the recipe will be revised when the
reference kernel library exists and b2 measures its honest-noise floor.
Substantial open questions are listed honestly (MoE routing, speculative
decoding, KV-cache audits, Apple Silicon achievability, closed-weight
models, verifiability-tax economics).

### Why both, why now

Implementation cannot start with byte-level ambiguity, and it cannot start
without a canonical numerics specification. Adding these now means: when the
first engineer opens an editor to write the reference client, every
primitive choice is pinned and the audited-path recipe is at least
provisionally specified. Six months of "we'll figure it out as we go"
become six months of writing code against a spec — different problems,
different costs, much better outcome.

The two RFCs together close BLOCK B1 (canonical numerics) and B3
(attestation wrapper format choices, via §1.1 CBOR + §3.4 key hierarchy),
and all the GAP items from the criticality review (PRF, hash, signature,
encoding, NCS encoding, test-vector format). The remaining BLOCK items —
B2 bond accounting model, B4 bootstrap sequence, B5 embedding model
versioning governance, B6 fork-recovery bond circularity — are smaller
and addressed via amendments to the existing design RFCs.

A fresh corpus-wide criticality re-check is the natural next step after
these land — to surface incompatibilities that the new primitives may
have introduced between what the design RFCs *assume* and what the
references *actually specify*.

---

## RFC-0008 · v0.1 → v0.2 · 2026-05-20 · Context strings, drand failover, BLS/Ed25519 transition, constants

A focused amendment closing the BLOCK and HARD items from the post-0008/0009
criticality re-check that this document is responsible for.

### B-NEW-3 — Logit-trace Merkle context strings

**Was:** §4.1's reserved-context table covered fault proofs
(`ants-v1-fault-merkle`) but not the commit-at-send Merkle that RFC-0003
§commit-at-send specifies. An implementer of RFC-0003 had `leaf_i =
H(dist_i ‖ i)` from the spec but no derive_key context to plug into BLAKE3.

**Now:** §4.1 adds `ants-v1-logit-trace-leaf` and `ants-v1-logit-trace-root`
contexts, plus `ants-v1-cache-entry-hash` for `Y_canon` content addressing
in RFC-0002 v0.2, plus `ants-v1-vrf-seed-degraded` for the fallback below.

### B-NEW-4 — Drand failover

**Was:** §4.2 bound the L2 VRF seed to drand. Real drand outages would
stall the chain entirely.

**Now:** new §4.3 specifies a degraded-seed fallback. 30-second timeout to
fetch the drand round; on timeout the proposer constructs the seed from
`BLAKE3.derive_key("ants-v1-vrf-seed-degraded", prev_block_hash ‖ height)`
alone and sets `degraded_seed: true` in the block header. Blocks finalise
normally; the degraded-seed flag is visible in chain history forever.

**Honest residual:** during outages, an attacker with 1/3 of the previous
committee can grind toward favourable next-committees. Security regression
bounded by outage duration. The fallback prevents stall, not all attacks —
the honest framing of what we can and cannot promise.

### B-NEW-5 — BLS/Ed25519 transition at bootstrap K

**Was:** §3.3 mandated BLS12-381 aggregation for all PoUH committee
signatures regardless of K. At bootstrap K=5–16 this is wasteful: BLS
per-verify is ~10ms vs ~0.05ms for Ed25519, and aggregation savings are
negligible for small K.

**Now:** §3.3 transitions based on K. At **K ≤ 16** committee members sign
individually with Ed25519 (block header carries K Ed25519 sigs + member
IDs). At **K > 16** BLS aggregate signatures apply as before. The K=16
threshold is calibrated for ≥3× savings at crossover and is itself
calibratable (`BLS_TRANSITION_K` in §7).

### H-NEW-1 — μ_0 recipe-dependent

**Was:** §7 had a single `MU_0_HONEST_NOISE_FLOOR` constant marked "TBD on
b2". RFC-0009 v0.1 implied μ_0 differs by recipe (≈ 0 for INT8 canonical,
~10⁻⁵–10⁻⁴ for FP16 fallback).

**Now:** §7 splits into `MU_0_INT8_CANONICAL` (≈ 0, the q24 quantisation
floor) and `MU_0_FP16_CANONICAL` (TBD on b2). Implementers no longer
conflate the two regimes.

### Cross-RFC consistency nits

§3.1 now explicitly says Tier 3 verification committees (RFC-0003, N=3–7)
use Ed25519 individual signatures, not BLS. §7 adds: `CHOKE_LOOP_INTERVAL`
(10s from RFC-0001, previously missing); bond formula multipliers
(`POUH_BOND_C_COMMITTEE`, `BOND_RISK_MULT_CACHE_WRITE`,
`BOND_RISK_MULT_SETTLEMENT`, all TBD on b2 — referenced by RFC-0004 v0.3
§Bonds); `BLS_TRANSITION_K` (16); `DRAND_TIMEOUT` (30s).
`BOND_DISPUTE_WINDOW` is reaffirmed at 7 days (one week) for consistency
with RFC-0004 v0.3's "default starting point: one week" phrasing.

### What this does not close

Three BLOCK items remain open after this amendment: B2 (bond accounting
model — how A is measured, frozen, decayed during bond hold), B4
(bootstrap sequence), B6 (fork-recovery bond circularity). They are
addressed in Amendments C and D following this one.

---

## RFC-0004 · v0.3 → v0.4 · 2026-05-20 · Bond accounting model + fork-recovery bond fix

Closes B2 and B6 from the pre-implementation criticality review — the two
remaining BLOCK items in the reputation layer.

### B2 — Bond accounting model

**Was:** RFC-0004 v0.3 §Bonds said "lock a bond of A" without specifying how
A is measured, frozen, decayed during the hold, composed across overlapping
acts, or admitted atomically. The bond mechanism was conceptually clear but
operationally underspecified — an implementer had to guess on every detail.

**Now:** new §Bond accounting model in RFC-0004 v0.4. Specifies:

- `A` is locally recomputable from counterparty-countersigned receipts (same
  primitive as tenure `T`), encoded as `u64 μNCS` per RFC-0008 §6.
- Bondable `A` is total `A` minus the sum of currently-locked bonds.
- Bond admission is atomic and signed by the verifier, propagated through
  L1 alongside fault proofs (so other peers see the lock).
- Bonds are frozen at their admitted value for the hold duration — they do
  *not* decay during the hold.
- On release, the bond returns to the pool as if it had been a fresh
  contribution at the *admission* time (preventing decay-clock laundering
  by repeated lock/release cycles).
- On slash, the bond's value is added to the standard slash event.
- Multi-act composition is additive: parallel bonds sum.
- Verifier responsibility: admit bonds only against verified `bondable_A`
  headroom; misadmission is an attributable fault and triggers the
  standard L1 slash mechanic against the misadmitting verifier.

The intended consequence is that high-stakes participation is naturally
limited by recent honest contribution — a peer with low recent A has low
multi-act capacity, regardless of accumulated tenure. The market clears on
contribution, not on accumulated capital. This is the same property the
(A, T, κ) spine was designed for, now applied to multi-act allocation.

**Why:** the bond mechanic is the core defence against single-decisive-act
defection. Without an implementable accounting model, the mechanic exists
in name only. v0.4 makes it implementable in code, reusing the (A, T, κ)
spine primitives rather than introducing new ones — small surface area
discipline.

### B6 — Fork-recovery bond circularity

**Was:** the v0.3 bond formula table said the fork-recovery vote bond was
"total T at stake in the recovery". But during a fork-recovery event the
chain itself is in dispute — "total T" is computed against *which* chain?

**Now:** the table entry explicitly references the pre-fault agreed state
(per §Fork-recovery sub-problem 2). The same timestamp that legitimises the
fork (the attributable-fault proof's signed timestamp) also defines the
state from which the bond is computed. The circularity dissolves through
the existing fork-recovery construction — no new mechanism needed, just
explicit cross-reference between §Bonds and §Fork-recovery.

### What this does not close

B4 (bootstrap sequence) remains, addressed in RFC-0010 (Amendment D
below). After C+D, all BLOCK items from the pre-implementation criticality
review plus all five BLOCK items from the post-0008/0009 re-check are
closed.

---

## RFC-0010 · added · 2026-05-20 · Bootstrap Sequence

A new RFC closing B4 from the pre-implementation criticality review — the
last remaining BLOCK item, and the one without which `main()` of the
reference client has no defined behaviour.

### Why this is a new RFC, not an amendment

B4 is genuinely cross-cutting. It touches every existing RFC — identity
attestation flow, DHT join, chain genesis, cache participation, verification
tier availability, economy initialisation, bond mechanism eligibility.
Spreading the bootstrap spec across all six design RFCs would have made
every one of them longer without making bootstrap itself any clearer. A
dedicated RFC concentrates the answer in one place.

### What it specifies

- **Genesis state**: the CBOR-encoded structured object every peer must
  agree on bit-for-bit, signed by all trustees at launch, hashed with
  BLAKE3 to produce the network identity. Covers: protocol version,
  embedding model hashes, trustee Ed25519 / BLS pubkeys + attestations,
  trustee decay rate, drand network and initial round, bootstrap DHT
  seeds, initial calibratable constants, launch timestamp, signature.
- **First peer's flow**: 13-step concrete sequence from TEE attestation
  through DHT join, L1 CRDT sync, L2 chain sync, cache participation,
  drand subscription, and entering steady-state participation. With
  failure modes specified (genesis hash mismatch, TEE attestation
  failure, bootstrap DHT timeouts).
- **Four phases**: tiny (1–9 peers), small (10–99), growing (100–999),
  mature (1000+). Each phase has different capability availability,
  different security assumptions, different governance posture.
- **Capability matrix**: a table the reference client consults to
  enable/disable features per phase. Tier 2 audit becomes available at
  N≥20; Tier 3 at N≥50; PoUH BLS transition at K=16; etc.
- **Self-securing condition**: mathematical statement of when the
  network no longer depends on trustee honesty for security.
  Approximately `N ≈ 200–500` peers active for 3–6 months under nominal
  parameters.
- **Reference launch sequence**: the concrete recipe for "v0.1 mainnet
  launch" — trustee selection, GenesisState assembly, drand integration,
  embedding model packaging, public release, 30-day monitoring targets,
  6-month sunset checkpoint per GOVERNANCE.md.

### What this does not close

RFC-0010 is itself early-draft. Specific calibrations (`δ_genesis`, exact
phase-transition thresholds) are b2-class testnet measurements. Cold-start
for late joiners, bootstrap DHT seed liveness, trustee key rotation during
bootstrap, off-network bootstrap — all flagged as open. Reference launch
sequence is recipe-level but actual launch remains a future event the
GOVERNANCE.md process must convene.

### The state after C+D

All six BLOCK items from the pre-implementation criticality review (B1
canonical numerics, B2 bond accounting, B3-partial attestation wrapper, B4
bootstrap, B5 embedding governance, B6 fork-recovery bond circularity) are
closed. All five BLOCK items from the post-0008/0009 re-check (B-NEW-1
through B-NEW-5) are closed. HARD and EDGE items from both reviews are
either closed or explicitly tracked in the open-questions sections of the
relevant RFCs.

The spec corpus is internally complete enough for the all-in-one reference
client implementation to begin — and for an external review of the whole
project to be a meaningful exercise rather than a cataloguing of known
gaps.

---

## RFC-0004 · v0.4 → v0.5 · 2026-05-20 · Round 1 — closures from multi-persona review

Closes four BLOCK items surfaced by the external multi-persona review of
2026-05-20 (`REVIEW-multi-persona-2026-05.md`, 7 specialist voices). All four
are scoped to RFC-0004; this amendment is a single RFC version bump with
substantive additions.

### N1 — G-Set pruning + late-joiner protocol

**Was:** the CRDT G-Set was append-only by design (security-correct,
monotone), but unbounded growth makes the network unsustainable at scale
(~18M proofs after 5 years at 10⁴/day; every peer retains the lot).

**Now:** new §G-Set pruning sub-section under §Layer 1 specifies
post-epoch-confirmation pruning. Once L2 epoch `E_n` is finalised AND
`E_(n+1)` confirms it, proofs included in `E_n`'s Merkle root become
eligible for L1 pruning. The L2 chain becomes the canonical record from
that point; archive nodes serve on-demand retrieval of pruned proofs.
Late-joiner protocol explicit: sync L2 (bounded), sync recent L1
(bounded), query archive nodes for specific historical proofs as needed,
verify against L2 Merkle root.

**Why:** the one-honest-path property is preserved at the active-epoch
level. For pruned proofs it becomes any-archive-node-honest-path —
different in mechanism but equivalent in security guarantee, because
proofs remain self-authenticating and the L2 Merkle root commits to
their existence. Pruning does not weaken `VERIFY`; it moves where the
proof lives.

### N2 — Partition recovery with equivocation slashing + Σ-T fork choice

**Was:** under network partition with 2/3 finality per epoch, two
partition halves could in principle both achieve 2/3 finality on
different blocks — the problem that has broken Solana repeatedly.
Recovery was an undefined open question.

**Now:** new §Partition recovery sub-section under §Layer 2 specifies
three composable mechanisms:

- **Equivocation slashing**: a validator signing conflicting blocks at
  the same height is committing self-authenticating fault (the two
  signatures are the proof). Terminal-slash applies. With K=64,
  achieving conflicting 2/3 finality requires ≥43 equivocating
  validators simultaneously — the deterrent is the design's primary
  defence.
- **Σ-T fork choice**: when two finalised chains nonetheless emerge,
  reconciliation uses cumulative tenure (Σ T) of validators in each
  fork. Tenure is rate-capped by κ, so Σ T is a robust proxy for
  "which fork has the more legitimate validator set." Losing fork's
  blocks revert; bonds/slashes/findings recorded only in the losing
  fork are reissued through the winning fork's subsequent epochs.
- **Social-Schelling fallback**: balanced partitions where both forks
  have `Σ T > θ · Σ T_total` fall back to §Fork-recovery — the named
  irreducible coordination assumption.

**Why:** the "chain is witness, not judge" property holds — partition
recovery does not require the chain to create slashes (equivocation
proofs are self-authenticating L1 fault proofs that exist regardless
of which fork wins). The chain's role is providing Σ T accounting and
the canonical reconciled history. The Σ T metric uses the slow,
hard-to-forge component of reputation, so a fork built on
freshly-rented cloud-TEE Sybils loses to a fork built on established
peers.

### N6 — Selective disclosure of receipts (formally specified)

**Was:** RFC-0004 v0.4 §Bond accounting model referenced "RFC-0003-b's
selective-disclosure" twice. RFC-0003-b did not exist as a document; it
was a working nickname from an earlier session that never crystallised.
The external reviewer correctly identified this as a broken
cross-reference.

**Now:** new §Selective disclosure of receipts as a first-class section
of RFC-0004 (where the receipt-based (A, T, κ) computation lives). It
specifies:

- The receipt-bag **Merkle tree** with `bag_root` committed on demand.
- The selective-disclosure protocol: peer reveals minimum subset +
  Merkle inclusion proofs; verifier verifies each receipt, the Merkle
  proofs, and the accumulated `Σ decayed_value ≥ b`.
- **Soundness via lower-bound property**: selective disclosure can only
  *under*-claim — fake receipts fail verification by countersignature,
  and partial subsets can only show less than the peer's true `A`.
- The **positive/negative asymmetry** that makes the mechanism safe:
  positive evidence is peer-controlled (can only under-claim); negative
  evidence is L1 non-suppressible (faults reach verifiers independently
  of what the peer reveals).
- **Compact summaries** for large peers: signed time-bucket-totals,
  audited by random selective opens; bucket granularity calibratable.
- Explicit acknowledgement that ZK selective disclosure is future work
  (door is open in §What we have not figured out yet).

The two prior "RFC-0003-b" references are updated to point to this
section. The status note and the new section's intro paragraph retain
the historical reference for traceability.

**Why:** N6 was a documentation bug as much as a missing mechanism.
Fixing it in RFC-0004 (not RFC-0003) keeps the (A, T, κ) spine + bond
accounting + selective-disclosure machinery in one place. The
positive/negative asymmetry is the same one the trust-closed
architecture rests on at the L1 level, applied here at the receipt
level — a satisfying structural reuse, no new primitive.

### Bond admission race condition — race-safe protocol without clock sync

**Was:** v0.4 §Atomicity said "the first to acquire bond admission
wins; the second fails" — but "first" was ambiguous without specifying
clock synchronisation across verifiers. Naive NTP-based ordering is
fragile to clock attacks and adds an infrastructure dependency the
protocol otherwise does not need.

**Now:** §Atomicity specifies a race-safe protocol:

1. Verifier signs `bond_admission(act_id, peer, b, V_id, L1_view_hash)`
   and gossips it to L1.
2. Verifier waits a propagation delay `δ_admission` (default 5s for
   time-sensitive acts; longer for slower acts).
3. On observed conflict, **deterministic tie-break** via
   `BLAKE3.derive_key("ants-v1-bond-tiebreak", epoch_seed ‖ act_id ‖ V_id ‖ admission_round)`
   — smaller hash wins; loser gossips `bond_admission_null`.
4. No conflict observed → admission final.

The protocol requires only loose clock sync (within `δ_admission`); no
NTP dependency. The L2 chain provides the `epoch_seed` that anchors the
tie-break function, so a peer cannot precompute favourable admission
timings to game the function.

**Why:** the L2 chain already provides the strong coordination object
verifiers need (`epoch_seed`). Using it for tie-break closes the race
deterministically without introducing new infrastructure, and is honest
about what loose-coupling the protocol can promise (epoch-level
agreement, not wall-clock-level).

### State after Round 1

Four of the six new findings from the multi-persona review are closed.
Two remain as **Round 2** work:

- **N3** — formal model of cross-layer composition (TLA+ or ApPCS),
  probably as RFC-0011.
- **N4** — implementation roadmap (`IMPLEMENTATION.md` at root,
  decomposing the all-in-one client into ~15 sub-components with
  dependency graph and realistic 24-36 month team-of-4-6 timeline).
- **N5** — RFC-0007 TSC mechanics to be drafted during v0.x, not
  forthcoming after v1.0.

Three EDGE-tier register corrections (open-hardware TEE timeline,
producer royalty 50%→60-70%, CoC enforcement not BDFL-only) are
Round 3.

The reviewer's named residuals — patient state-actor, censorship at
L2, vendor compromise, all empirical parameters pending b2 — remain
acknowledged as irreducible or testnet-dependent.

---

## Round 5 — Language alignment: reference client in C · 2026-05-20

A foundational decision on the reference-client implementation language,
recorded here because it touches two spec documents (RFC-0009 and
IMPLEMENTATION.md) and is large enough to warrant a CHANGELOG entry of
its own, but small enough not to be a full review round.

### The decision

The reference client is implemented in **C (C99/C11)**, not Rust. The
choice was made on the basis of two priorities the founder articulated
when the conversation pivoted from spec to code: **maximum performance**
on the audited path, and **total cross-platform reach** (every server,
every Apple Silicon, every Android NDK target, every embedded ARM chip
with a TEE — *la rete deve essere pura*).

### What the choice gains

- **Performance at the metal.** SIMD intrinsics, memory layout control,
  cache-line awareness — C is the default tooling for all three, with 30
  years of compiler maturity. The canonical kernels (component #12) sit
  directly on `ggml` (which is C, and is the state-of-the-art for
  quantized inference on commodity CPUs in 2026).
- **Total platform reach.** Every TEE SDK (Intel SGX/TDX, AMD SEV-SNP,
  ARM CCA, Apple Secure Enclave, Qualcomm QSEE) is C/C++ native — no
  binding work for the largest foundation component (#3 TEE harness,
  6 EM). C compilers exist on every chip that matters, including
  embedded targets (Cortex-M, RISC-V) the protocol may want to support
  later.
- **ABI stability and audit-friendliness.** C ABI has been stable for
  decades; the entire ecosystem can link against C libraries across
  compiler versions. Every line of C does one thing — no hidden
  destructors, no template expansion, no constructor side effects.
  Audits of crypto and consensus code are easier on C than on languages
  with richer semantics. Bitcoin Core's mixed C/C++ codebase is the
  closest operational precedent; `ggml` and `secp256k1` are the pure-C
  precedents that demonstrate the approach scales.
- **No FFI boundary.** Crypto (BLAKE3, `blst`, `ed25519-donna`),
  inference (`ggml`), CBOR (`tinycbor`/`nanocbor`), TEE SDKs — all C
  native. The reference client links directly without wrapper layers.

### What the choice costs (named honestly)

- **No compile-time memory safety.** Use-after-free, buffer overflow,
  data race classes that Rust would catch at compile time are runtime
  concerns in C. Mitigation strategy: discipline (defer/cleanup macros,
  arena allocators for hot paths), static analysis with Clang
  `-Weverything`, AddressSanitizer + ThreadSanitizer + UBSan in CI,
  fuzzing via libFuzzer/AFL++ on every protocol-parser surface,
  external security audit at Phase F (per IMPLEMENTATION.md). Manifesto
  Thesis 18's "We Acknowledge" register applies here as well: the
  trade-off is accepted, not hidden.
- **More code written by hand.** No `std::vector`, no `std::map`, no
  RAII. Realistic increase: ~30–50% more lines of code vs C++17, much
  of which is cleanup/error-handling discipline. Every line is
  obvious; the audit benefit compensates.
- **No `cargo`-style unified tooling.** Build is **CMake superbuild**
  (cross-platform) or pure Make + autoconf (purist). Dependency
  management is per-platform package managers (apt, brew, vcpkg) or
  vendored sources (`deps/ggml/`, etc.). The friction is real but
  manageable; the entire C ecosystem has done it for decades.

### What changed in this entry

- **RFC-0009 v0.4 → v0.5** (early). §"The recipe is not implemented
  yet" updated: kernel library is in C99/C11 with hand-tuned SIMD
  intrinsics + assembly, expected to leverage `ggml` as computational
  foundation, with the discipline layer (left-biased tree reductions,
  divide-not-reciprocal scale arithmetic, q24, INT8-GPTQ-128) above
  it. §"The reference kernel library" skeleton rewritten as a C
  library structure (`.c`/`.h` files, `CMakeLists.txt`, `deps/ggml/`
  vendoring/submodule, `cpu_scalar.c` as bit-exact reference).
- **IMPLEMENTATION.md** new §"Language and stack" section between
  §"Why this document exists" and §"The fifteen sub-components".
  Documents the C choice, names the reference library baselines for
  each layer (`ggml`, BLAKE3, `blst`, `ed25519-donna`, ECVRF-ELL2,
  `tinycbor`/`nanocbor`, TEE SDKs, CMake), and records the repo
  layout (`ants-client` + `ants-test-vectors` sibling repos to
  `ants` spec repo) and licence choice (Apache-2.0 OR MIT
  dual-licence) per the Round 5 decisions.
- **IMPLEMENTATION.md component table** descriptors updated to
  reflect C choice: #1 crypto primitives mentions specific C
  libraries; #2 CBOR codec names `tinycbor`/`nanocbor` candidates;
  #3 TEE harness notes "all C/C++ native, no binding work"; #12
  canonical kernel library explicitly says "C library, leverages
  `ggml` as computational foundation."

### What did not change

- **No protocol behaviour changes.** No constant moved. No security
  argument was strengthened or weakened. The corpus continues to
  specify *what* the protocol does and *why*; the language is a *how*
  decision and lives in IMPLEMENTATION.md plus the kernel-library
  skeleton.
- **No new RFC version bumps except RFC-0009.** RFC-0008's
  language-agnostic primitives spec is unchanged. The reference
  client could be reimplemented in any other systems language and
  still conform — the bytes on the wire are the spec, not the
  source.
- **No effort estimate changes.** 72–92 EM remains the figure;
  IMPLEMENTATION.md's 24–36-month wall-clock figure is unaffected.
  C is comparable to Rust in raw effort once the binding work for
  TEE SDKs and ML libraries is accounted for (Rust would have saved
  on memory-safety review but spent on FFI).

### State after Round 5

The corpus is implementation-language-aligned. Next operational step
per the founder's decision is creating `Ants-Community/ants-client`
and `Ants-Community/ants-test-vectors` repositories with the agreed
licence and CMake scaffolding, then opening `[CLAIM]` issues on
`Ants-Community/ants` for the foundation-layer components (#1
crypto primitives, #2 CBOR codec, #3 TEE harness). Code starts
after the nomination window per IMPLEMENTATION.md §"How to claim a
sub-component."

The spec corpus remains review-cycle-complete from both
multi-persona reviews. This entry records a decision about *how*
to build, not *what* to build.

---

## Round 4 EDGE — register corrections from multi-persona Round 2 review · 2026-05-20

Twelve register-level corrections from the Round 2 multi-persona review.
Recorded in one entry because each is a documentation/clarity fix, not
an architectural change. Where a new constant or constraint follows
from the correction, it is listed.

### MANIFESTO — new Thesis 18 (We Acknowledge)

**Added.** New thesis 18 under a new §"We Acknowledge" section: *"We
depend, today, on hardware we did not build."* Names the hardware-vendor
trust the protocol relies on for TEE attestation in 2026, commits to
migrating to open-hardware TEE roots when production-grade (2030–2032),
and explicitly works toward the day the thesis is obsolete. The
manifesto previously had only positive ("We Build") and negative ("We
Refuse") theses; this adds an honest residual ("We Acknowledge") to
the structure.

**Why:** the TEE-security persona of the Round 2 review noted that the
corpus loads RFC-0005 with increasing responsibility while RFC-0005
itself depends on closed-vendor TEE attestation we cannot eliminate.
The manifesto previously did not explicitly acknowledge this — the
acknowledgment thesis closes the rhetorical gap.

### GOVERNANCE — new §"Voting weight, attested identity, and the
'datacenter vs freelancer' caveat"

**Added.** New section between §"Structure beneath the BDFL" and §"How
decisions are made". Makes explicit the consequence of the 1-CPU-1-voice
attestation rule: a university with 20 attested machines votes 20 times
in elections that count by attested identity; a freelancer with one
laptop votes once. Names the three mitigations the protocol already
provides (tenure-gated influence, uniform VRF selection within
committees, credible-fork-threat) without claiming to eliminate the
proportionality.

**Why:** the OSS-governance persona observed that "1 CPU = 1 voice"
combined with hardware-attestation Sybil resistance produces a voting
weight tied to compute resources, which is worth declaring explicitly
rather than letting it sit as an implicit feature/bug debate.

### RFC-0002 v0.3 → v0.4 — verifiability-tax / royalty separation

**Was:** §"Integration with the community economy" stated the 65/35
royalty split and the verifiability tax separately but did not address
the misreading that they compose at the producer's detriment.

**Now:** new sub-section §"The verifiability tax and the royalty split
do not compound at the producer's detriment" specifies the three
charges on different axes: the tax is paid once by the producer at
fresh inference; the royalty split is paid by the consumer on every
retrieval; Tier 2/3 verification fees are paid by the requester to
verifiers. Net effect: nobody is paying the verifiability tax at
retrieval time.

**Why:** the game-theory persona's reading was a misreading, but a
misreading the spec should preclude. The clarification removes the
ambiguity.

### RFC-0005 v0.1 (early, Round 4 EDGE amendments)

Three Round 4 EDGE additions to RFC-0005:

- **New §"The 2028–2030 vulnerability window"** under §"Sybil
  economics". Names explicitly the period where all three long-term
  defences (network growth, open-hardware migration, (A, T, κ)
  amortisation) are simultaneously weak. Three operational mitigations
  named: foundation as first responder per Manifesto Thesis 16,
  GOVERNANCE.md sunset checkpoints attending to the window,
  credible-fork-threat as the irreducible coordination floor.
- **New §"Historical TEE vulnerabilities and the protocol's response
  posture"**. Catalogues recurring TEE failure modes (Foreshadow,
  Plundervolt, SGAxe, ÆPIC Leak, Downfall, Inception, CacheWarp,
  Heckler, cross-VM Spectre) with a structured response posture: no
  protocol-version bump required to absorb a new disclosure;
  multi-vendor weighting contains the blast radius; vendor revocation
  timing is bounded.
- **§"Vendor compromise" extended with vendor revocation propagation
  timing.** New constant `VENDOR_REVOCATION_PROPAGATION_BOUND`
  (RFC-0008 §7, default 72 hours). Revocations propagated as L1 CRDT
  objects; after 72 hours from first observation by any honest peer,
  revoked attestations are rejected regardless of personal observation.

**Why:** the TEE-security persona's three concerns (window of weakness,
absent vulnerability catalogue, undefined revocation timing) all
needed explicit treatment rather than implicit hope. RFC-0005 remains
v0.1 (early) — these are register/documentation additions, not the
substantial revisions the "early" tag anticipates.

### RFC-0007 v0.1 → v0.2 — three additions

- **§"Off-cycle removal" extended with size-scaled petition threshold.**
  Replaces the static 25% rule with
  `max(PETITION_FLOOR, min(PETITION_CEILING, ⌈0.25 · N_eligible⌉))`,
  defaults floor=12 and ceiling=150. Three regimes (small-network
  floor / linear / large-network ceiling) avoid both "5 friends can
  petition" at small N and "2,500 signatures impractical" at large N.
- **§"The chair seat" extended with the agenda-setting authority
  acknowledgment.** Names explicitly that the transitional BDFL-as-chair
  retains agenda-setting authority for 24 months post-v1.0, with three
  structural bounds (chair-removal threshold, public-agenda discipline,
  hard time limit). The BDFL does not disappear at v1.0 declaration;
  the BDFL disappears 24 months later.
- **New §"Catastrophic transition scenarios"** specifying continuity
  protocols for four expected failure modes: S1 transitional chair
  incapacitation, S2 multiple-seat vacancy, S3 contested first
  election, S4 persistent TSC dysfunction. Each scenario has named
  continuity mechanics that do not require ad-hoc improvisation
  mid-crisis.

**Why:** the OSS-governance persona's three concerns (petition scaling,
BDFL chair tail, transition stress test) all needed structural rather
than aspirational answers. The transition-stress-test section is
particularly important because the failure scenarios are *expected*,
not pathological, in a 7-person body with zero operational precedent.

### RFC-0008 v0.4 → v0.5 — two clarifications + two new constants

- **§3.3 §"Disambiguation of K"** clarifies that `K` is the
  VRF-selected committee size (not the effectively-signing count); the
  scheme transition (Ed25519 ≤ 16 / BLS > 16) is **epoch-atomic** —
  every block within an epoch uses the same scheme determined by the
  epoch's `K`; the proposer does not choose. Offline members do not
  change `K` for the epoch; they just reduce the effectively-signing
  count, and 2/3 finality computes against `K`.
- **§4.3 §"Quantifying the grinding compounding bound"** quantifies the
  worst-case attacker committee-share growth during prolonged drand
  outage: `attacker_share(N) ≈ baseline + Δ · sqrt(N)` for `N`
  consecutive degraded blocks, with sub-linear (sqrt) compounding.
  Concrete table: 1h outage ~41% share, 24h outage ~71% share, 72h
  outage ~98% share. New constant
  `DRAND_OUTAGE_EMERGENCY_THRESHOLD` (default 6 hours) triggers an
  emergency-response posture.
- **§7 reference constants table** adds `VENDOR_REVOCATION_PROPAGATION_BOUND`
  (72h, supports RFC-0005) and `DRAND_OUTAGE_EMERGENCY_THRESHOLD`
  (6h, supports §4.3 grinding bound).

**Why:** the applied-cryptography persona's two concerns (K transition
ambiguity, drand grinding compounding bound) were specification gaps
that needed precise fills, not approximate gestures. The K
disambiguation reveals that what looked like ambiguity was a missing
sentence about epoch-atomicity; the grinding bound becomes a quantified
table with a triggered emergency threshold.

### RFC-0010 v0.3 → v0.4 — unit clarification

**Added.** Unit note in §Admission rules clarifying that "epoch" refers
to `EPOCH_DURATION` (24h), not `POUH_BLOCK_TIME` (30s). The 7-epoch
default `ROTATION_ADMISSION_WINDOW` is therefore ≈1 week wall-clock,
not 3.5 minutes block-clock. Drand outages do not collapse this window
since epochs continue to advance via degraded-seed fallback.

**Why:** the cold-eyed software engineer persona's calculation
("7 epochs at 30s blocks = 3.5 minutes") confused epoch with block.
The clarification is unit-level register correction; the underlying
mechanism is unchanged.

### State after Round 4 EDGE

The 12 EDGE items from the multi-persona Round 2 review are closed.
The remaining Round 2 "ack-only / operational" items (TLA+ contributor
identification, foundation funding, MoE/SpecDec/long-context
canonical-recipe extensions, CBOR library audit, test vectors
chicken-egg, etc.) are downstream of spec work — operational and
funding decisions, not specification gaps. The corpus has now
absorbed every reviewable concern from both review rounds.

What remains, restated honestly: empirical b2 calibration of σ, e,
T_prop, w, κ, δ_A, δ_T, T_CAP, w, T_FORK_CHOICE_CAP, R_ARCHIVE_MIN,
EMERGENCY_REVOCATION_WINDOW, ATTESTATION_FRESHNESS_WINDOW,
VENDOR_REVOCATION_PROPAGATION_BOUND, DRAND_OUTAGE_EMERGENCY_THRESHOLD,
PETITION_FLOOR, PETITION_CEILING — all calibratable on testnet
evidence — plus the irreducible residuals handed to the
credible-fork-threat. The corpus is review-cycle-complete for what
the corpus itself can specify.

---

## Round 4 HARD — closures from multi-persona Round 2 review · 2026-05-20

Five architecturally meaningful closures from the Round 2 multi-persona
review (`REVIEW-multi-persona-2026-05-round2.md`), each addressing a
new concern arising from the Round 1+2+3 fixes themselves. The Round 2
review explicitly framed: "now that the design is filled in, does it
survive its own additions?" — these are the five places where the answer
needed sharpening before continuing.

### RFC-0004 v0.5 → v0.6 (1) — Saturating Σ T_eff for fork choice

**Was:** §Partition recovery §Fork choice by Σ T used raw `Σ T` as
the metric for reconciling competing finalised chains.

**Now:** §Fork choice by Σ T_eff applies a saturating transform
`T_eff(T) = T_CAP · (1 − exp(−T / T_CAP))` to `T` *for fork-choice
purposes only*. Properties: linear regime for young peers, saturating
regime for tenured peers, monotone+smooth+deterministic, no threshold
cliff. The cap applies **only** to fork-choice — raw `T` still governs
verifier-set eligibility, bond capacity, persistence, and the
slow-decay floor. New constant `T_FORK_CHOICE_CAP` added to RFC-0008
§7 (default ≈ 2 years of κ-bound tenure, b2-calibrated).

**Why:** the DS-systems persona and the game-theorist persona of the
Round 2 review *independently* identified the same long-tail concern:
raw `Σ T` makes a 10-year peer weigh 10× a 1-year peer, so fork choice
in a mature network is dominated by historical peers — "founder weight
rebranded as tenure dominance," in literal tension with the
manifesto's authority-decay arguments and with RFC-0010's δ_genesis
sunset. The saturating transform preserves the metric's robustness
against fresh-Sybil forks (the original purpose) while preventing the
metric from collapsing into pure age-weighting in the long run.
Capping `T` in non-fork-choice contexts would punish exactly the
long-term honest contribution the spine is designed to reward — hence
the surgical scope.

### RFC-0004 v0.5 → v0.6 (2) — Archive nodes and the pruning-DoS defence

**Was:** §G-Set pruning §Archive nodes specified retention as an
"emergent optional role" with an unspecified NCS reward and no
minimum-redundancy requirement. The safe-direction-of-error rule
(late joiners default-treat as slashed when proofs cannot be
retrieved) combined with optional retention created an unaddressed
attack surface.

**Now:** §Archive nodes and the pruning-DoS defence specifies three
composable layers: (1) **minimum-redundancy target** `R_ARCHIVE_MIN`
(default 8) per pruned proof, monitored by the L2 committee as part
of each epoch summary's rarity flag; (2) **NCS reward scaled by
inverse redundancy** — a proof held by only 1 archive earns 8× the
baseline retrieval fee; (3) **passive late-joiner archiving** —
verified historical proofs are retained by default for at least one
epoch and may be retained indefinitely. `R_ARCHIVE_MIN` added to
RFC-0008 §7.

**Why:** the DS-systems persona noted that the safe-direction rule
"converts an impossible slash-fabrication attack into an isolation
attack" — an adversary withholding valid proofs forces late joiners to
default-slash. The fix raises the cost of that attack from "drop one
peer from archive duty" to "convince a quorum of archives + outpace
the reward-scaling + suppress passive new-joiner archiving" — the same
state-actor-residual posture the corpus takes everywhere else, now
explicit at the archive layer. The mechanism uses existing primitives
(L2 epoch summaries, gossip-observed hold counts, retrieval fees), no
new cryptography.

### RFC-0009 v0.3 → v0.4 (early) — Bit-exact per-token scale computation

**Was:** §3.2 specified per-token symmetric dynamic INT8 quantization
with the scale derived "deterministically from the input" — but did
not pin the recipe for the max-activation reduction or the scale
arithmetic itself.

**Now:** new §2.1 "Bit-exact per-token scale computation" pins four
steps: (1) absolute-value pass via scalar IEEE-754 or SIMD `vandps`
on the sign-mask (equivalent); (2) max reduction via a **left-biased
binary tree** over a power-of-two-padded array with `-∞` padding;
(3) scale by direct FP32 divide `max_abs / 127.0f`, **not** by
precomputed reciprocal multiplication; (4) explicit zero-token
sentinel to prevent platform-dependent NaN/Inf propagation. The
general rule applies to any FP32 value derived from input data and
used as a multiplicative factor downstream.

**Why:** the verifiable-computation persona correctly identified the
weakest seam in v0.3's bit-exactness argument: "the INT8 × INT8 →
INT32 accumulator is exact, but the *scale values* are runtime-derived
from activation data, and if two honest implementations compute
max-activation in different order, scales can differ in the last bits
and the downstream dequantize becomes non-bit-identical." The fix
costs essentially nothing on modern CPUs (left-biased tree is the
default of standard SIMD horizontal-max; divide-vs-reciprocal is a
one-instruction swap), but "essentially nothing" is not zero — and
the canonical recipe is the only place where the bit-exact contract
holds.

### RFC-0010 v0.2 → v0.3 — Equivocation freeze escape hatch

**Was:** §Equivocation said a slot frozen by a trustee equivocation
"resolves" only through a `TrusteeRevocation` with full witness
threshold. The standard `TrusteeRevocation` path required the
remaining trustees to coordinate witness acks in real time —
implicitly assuming synchronous availability across all jurisdictions
during a crisis.

**Now:** new §Escape hatch from the freeze without synchronous
coordination specifies an **asynchronous** ack accumulation protocol.
The first remaining trustee publishes a *provisional* envelope (no
`prev_sig`, reason `SUSPECTED_COMPROMISE`); other trustees may append
acks to it at any point within the extended
**`EMERGENCY_REVOCATION_WINDOW`** (default 30 epochs ≈ 30 days,
calibratable). The envelope becomes final the moment accumulated acks
reach `⌈2/3 · (T − 1)⌉`. Anti-equivocation discipline preserved:
a trustee signing two conflicting provisionals commits its own
equivocation. New constant added to RFC-0008 §7.

**Why:** the applied-cryptography persona identified the deadlock case:
trustee compromise + key loss + attacker equivocation locks the slot,
and the recovery path required real-time multi-jurisdiction coordination
during a crisis. The fix removes the synchronous-coordination
assumption without changing the quorum requirement. Safety (`⌈2/3 · (T
− 1)⌉` of remaining trustees still required) is preserved; what
changes is the coordination shape (acks arrive asynchronously, from
any jurisdiction, over a 30-day window). The honest framing: the trust
assumption did not weaken; the practical operating envelope of the
same assumption widened to match the global, multi-jurisdiction
reality of the trustee set.

### RFC-0005 v0.1 (early, Round 4 amendment) — Attestation freshness window

**Was:** §The attestation flow mentioned that attestations "have
expiration windows (typically a few weeks to a few months, depending
on vendor)" but did not define "fresh" precisely. RFC-0010 v0.2's
trustee-rotation §Admission rules required `new_attestation` to be
"fresh, valid TEE attestation per RFC-0005" — a circular reference.

**Now:** new §Attestation freshness window specifies
**`ATTESTATION_FRESHNESS_WINDOW`** (default 30 days, calibratable per
RFC-0008 §7) as a wall-clock comparison against the attestation's
signed `issued_at` timestamp, ±5 minutes clock-skew tolerance. The
window applies uniformly to every context in the corpus where
attestation is required: standard peer handshake, trustee key
rotation, bond admission, committee role assumption. "Fresh" now
means the same thing everywhere it appears.

**Why:** the TEE-security persona correctly flagged the circular
reference. The fix is the smallest possible: pick a window, pin a
clock-skew tolerance, apply uniformly. Thirty days is the rough
midpoint matching vendor revocation-propagation latency in 2026 and
honest-operator refresh cadence; calibratable on b2.

### RFC-0008 v0.3 → v0.4 (Reference)

Four new calibratable constants added to §7:

- `T_FORK_CHOICE_CAP` (≈ 2 years of κ-bound tenure)
- `R_ARCHIVE_MIN` (8 distinct holders per pruned proof)
- `EMERGENCY_REVOCATION_WINDOW` (30 epochs ≈ 30 days)
- `ATTESTATION_FRESHNESS_WINDOW` (30 days)

All Calibratable per the C/P distinction; non-breaking additions per §9.

### State after Round 4 HARD

The five "new concerns arising from the fixes" of the Round 2 review
are closed. The Round 2 review's twelve EDGE register-correction items
remain open (sequencing them next, in one tighter pass). The Round 2
"ack-only or operational" items (TLA+ contributor identification,
foundation funding, MoE/SpecDec/long-context canonical-recipe
extensions, etc.) are downstream of spec work and remain so.

The corpus continues to converge: each review surfaces a tighter and
smaller set of concerns, and each closure round reuses existing
primitives rather than introducing new ones. The Round 2 review's
bottom-line observation — that the project has shifted from "are we
specifying well" to "are we executing" — remains the correct framing
for the next phase.

---

## Round 3 — EDGE register corrections · 2026-05-20

The five EDGE-tier items from the multi-persona review of 2026-05-20,
addressed in a single amendment. None of these closes an architectural
gap; each corrects a register-level claim that a sharp reader would
otherwise rightly challenge. Recording in one entry because they are
register corrections, not design pivots.

### RFC-0005 v0.1 → v0.1 (early, two amendments)

**Was:** §"The long-term open-hardware path" claimed "Currently
planned: 2027–2028 for Keystone, OpenTitan acceptance." §"Sybil
economics" tabulated cloud-TEE costs at 2026 prices with no
treatment of price erosion over time.

**Now (timeline):** the realistic horizon is **2030–2032** for first
production-grade Keystone or OpenTitan acceptance. Open silicon,
audit, supply chain, and ecosystem maturity together take longer
than the research roadmap implies. The earlier 2027–2028 estimate is
named as optimistic; the corrected horizon is named as multi-year
"half-decade scale, not 2–3-year scale".

**Now (price erosion):** new §"Price erosion: the snapshot is not a
permanent floor" subsection. Confidential-compute pricing has
followed a roughly 15–25% real-terms annual decline since mainstream
availability; the protocol must plan for the 2026 snapshot to erode
on this trajectory. Projection table out to 2035 included with
explicit caveats (trend can flatten or reverse). The honest framing
recorded: hardware-attested identity is a **decade-class Sybil
deterrent, not a perpetual one**, and the long-term defence is the
stack of (a) network growth outpacing price decline, (b) the
open-hardware migration of the §"open-hardware path", (c) the
(A, T, κ) spine in RFC-0004 amortizing each identity with sustained
honest contribution.

**Why:** the previous text overstated both the migration speed and
the durability of the cost wall. The 2026 table is correct as a
snapshot but inert as a permanent claim. Stating the trajectory
preserves the deterrent's honest framing.

### RFC-0009 v0.2 → v0.3 — q24 collision and birthday-attack bounds

**Was:** §"The unembedding step" stated "q24 — 24 fractional bits"
and "7 decimal digits of precision, well above any honest
discrepancy" but did not quantify what an attacker can do with q24
collisions.

**Now:** new §5.1 "q24 collision and birthday-attack bounds" makes
the four relevant bounds explicit: single-logit collision 2⁻²⁴
(insufficient on its own; never committed in isolation); full-position
collision 2⁻²⁴ᵛ ≈ 2⁻⁷⁶⁸⁰⁰⁰ for V=32K (infeasible); Merkle-root
collision 2¹²⁸ (the actual cryptographic-strength bound, provided by
BLAKE3 not by q24); challenge-grinding closed by the beacon-posterior
challenge sub-protocol of RFC-0003 §"Challenge unpredictability".
q24 is named explicitly as the *value representation* that makes
honest convergence possible, NOT as the collision-resistance layer.
The widening path to q32 is reserved without exercising it.

**Why:** the previous "7 decimal digits" framing was correct but
under-specified the security argument. A reader could (and a
reviewer did) reasonably ask whether 24 bits per logit is enough
for collision resistance. The answer — "yes, because the
collision-resistance is at the Merkle root above, q24 only needs
to make honest implementations agree" — is now in the document.

### RFC-0002 v0.2 → v0.3 — Producer royalty 50% → 65% (60–70% band)

**Was:** §"Integration with the community economy" declared
"Producer royalty: 50% of cache-hit fee" with the symmetric host
share. §"What we have not figured out yet" noted the 50% as a
balanced guess.

**Now:** the producer share is **65%, with a 60–70% calibration
band**, and the host share is the symmetric 30–40%. The economic
rationale is named explicitly: at the protocol's early phase the
binding constraint is **specialist supply**, not host supply
(storage and bandwidth are commodified and abundant in the
addressable peer population). Tilting the split toward the producer
increases the long-tail incentive that makes the cache
architecturally valuable — the producer earns on every retrieval
over the entry's lifetime; the host earns on volume across
many entries. RFC-0008 §7 `PRODUCER_ROYALTY_FRACTION` updated to
"65% (60–70% band)".

**Why:** the symmetric 50/50 split treated the two sides as
interchangeable at parameter-setting time; the reviewer correctly
noted that the asymmetric supply elasticity argues for an
asymmetric split. The exact share remains a calibrated parameter,
revisable on b2 evidence; what changed is the default and the
explicit naming of which side the protocol believes is the
binding constraint.

### GOVERNANCE — CoC enforcement via 2–3 community moderators panel (not BDFL-only)

**Was:** §"Code of Conduct and enforcement" placed routine
enforcement on the BDFL with the option to delegate to "specifically
named community moderators, who report to the BDFL."

**Now:** routine enforcement is the responsibility of a **panel of
two to three community moderators**, named by the BDFL but operating
with day-to-day autonomy. The panel handles Matrix moderation,
GitHub triage, and first-line response (warnings, temporary mutes,
content removal, time-bounded suspensions). Serious incidents
(harassment, doxxing, coordinated bad-faith activity, or anything
the panel cannot resolve by consensus) escalate to the BDFL during
v0.x and to the TSC after v1.0. The BDFL retains a final-veto
authority over moderator decisions, used sparingly and recorded
with reasoning when used.

**Why:** single-point dependency on the BDFL is fragile (overworked
maintainers are poor first responders; backlog or rubber-stamp are
the realistic failure modes) and community legitimacy of
enforcement matters (a panel from the contributor body brings its
own standing). The change preserves BDFL escalation authority while
moving the first response to a panel that is structurally less
prone to either failure mode.

### State after Round 3

All review items from `REVIEW-multi-persona-2026-05.md` are now
closed: 4 BLOCKs (Round 1, commit `f6fe3bd`), 3 HARDs (Round 2,
commit `93ef697`), 2 Round-2 residuals (commit `aec1180`), 5 EDGEs
(this entry). Remaining items are testnet-empirical (b2-class
measurements: `σ`, `e`, `T_prop`, `w`, partition-recovery rehearsal,
royalty-share calibration, ROTATION_ADMISSION_WINDOW tuning, cache
hit volumes for the producer/host elasticity split) or explicit
irreducible residuals handed to the credible-fork-threat.

The named reviewers from the multi-persona review are worth crediting
in this CHANGELOG with consent at the point of public acknowledgement,
once that round is appropriate.

---

## RFC-0008 · v0.2 → v0.3 + RFC-0010 · v0.1 → v0.2 · 2026-05-20 · ECVRF constant-time + in-protocol trustee key rotation

Closes the two Round-2 residuals left out of the `93ef697` corpus: the
timing side-channel in the pinned VRF ciphersuite (RFC-0008), and the
trustee-key-rotation operational gap that GenesisState immutability
created (RFC-0010).

### RFC-0008 — ECVRF ciphersuite TAI → ELL2

**Was:** §4.2 pinned `ECVRF-EDWARDS25519-SHA512-TAI`. The
try-and-increment hash-to-curve iterates with a counter until a valid
curve point is found; the iteration count depends on the input. On
hardware with cache or branch-prediction side-channels, this leaks
information about the VRF input through the wall-clock timing of
`prove`/`verify`.

**Now:** §4.2 pins `ECVRF-EDWARDS25519-SHA512-ELL2`. The Elligator 2
hash-to-curve maps any field element to a curve point in a single
constant-time operation. Output distribution and verifiability are
identical to TAI; only the timing channel closes. Both ciphersuites
are documented in [RFC 9381](https://www.rfc-editor.org/rfc/rfc9381).

**Why:** the VRF inputs (previous-block-hash, height, drand round) are
themselves public, so the practical leak is bounded — but pinning a
primitive whose constant-time variant is freely available and equally
interoperable would have been gratuitous. The change is pre-v1.0 and
costs nothing on the wire. The pinning §9 framing for "breaking change
to pinned primitive = major version bump" applies post-v1.0; during
v0.x, primitive corrections of this kind are allowed and travel as
ordinary `vN.x → vN.(x+1)` amendments.

### RFC-0010 — Trustee key rotation in-protocol

**Was:** §"What we have not figured out yet" explicitly listed trustee
key rotation as undefined: "the GenesisState is supposed to be
immutable, so trustee key rotation effectively requires a
protocol-version bump." This left a realistic 6–12 month bootstrap
event — hardware replacement, attestation refresh, key compromise,
planned departure — without an in-protocol path.

**Now:** new §Trustee key rotation specifies an in-protocol mechanism
that does **not** require a protocol-version bump. Pieces:

- **Slot/key separation**: the GenesisState `trustees[]` array fixes
  trustee *slot* identity; the *key* bound to each slot is mutable.
  Granted-weight schedule remains anchored to slot creation time, so
  rotation cannot reset the sunset curve.
- **KeyRotationAnnouncement**: signed by the outgoing key + 2/3 of
  *other* current trustees as witness acks, carrying a fresh TEE
  attestation for the new key, propagated through L1 alongside fault
  proofs. Two new context strings (`ants-v1-trustee-rotation`,
  `ants-v1-trustee-rotation-ack`) added to RFC-0008 §4.1.
- **Forced rotation / revocation**: lost-key and compromise cases that
  cannot self-sign use `TrusteeRevocation` (same envelope, `prev_sig`
  omitted) requiring full 2/3 witness threshold, with the new key
  optional (slot may be left vacant).
- **Equivocation handling**: two conflicting announcements at the same
  slot/epoch produce self-authenticating fault; the slot freezes until
  a witness-quorum revocation resolves it.
- **New calibratable constant**: `ROTATION_ADMISSION_WINDOW` (default
  7 epochs ≈ 1 week) added to RFC-0008 §7.

**Why:** the immutability of GenesisState was load-bearing for *protocol
identity* (genesis hash, embedding model pinning, drand network), not
for the operational fact of which keys are currently bound to each
trustee slot. The mechanism makes that distinction precise and reuses
existing primitives (L1 CRDT, BLS, TEE attestation) — no new
cryptographic assumption. The compromise scenario reduces to the same
honest-majority assumption RFC-0004's L2 finality already makes; a
coordinated >1/3 trustee compromise hands off to the credible-fork-threat
as everywhere else.

### State after this amendment

The two Round-2 residuals flagged in the `93ef697` post-mortem are
closed. The remaining post-multi-persona-review items are the Round 3
EDGE-tier register corrections (open-hardware TEE timeline, q24
birthday-attack quantification, producer royalty 50→60-70%, CoC
moderators not BDFL-only, TEE price erosion in Sybil economics) and
the testnet-empirical b2-class measurements.

---

## Round 2 — RFC-0011 added + IMPLEMENTATION.md added + RFC-0007 promoted · 2026-05-20

Closes the three HARD items remaining after Round 1: the formal model
(N3), the implementation roadmap (N4), and the TSC mechanics (N5). Three
distinct deliverables, one CHANGELOG entry.

### N3 — RFC-0011 Formal Model of Cross-Layer Composition (added)

**Was:** the corpus had no formal model of how the four time-scales (DHT
eventually-consistent, CRDT monotone, chain 2/3 finality, drand
round-bounded) compose under adversarial conditions. Each design RFC
reasoned about its own layer; none reasoned about the composition.
Solana-class partition-recovery failures live at exactly those seams.

**Now:** RFC-0011 added at Draft v0.1. Specifies what a TLA+ (or
equivalent) artefact must capture: 10 invariants (slash safety/liveness,
CRDT convergence, chain finality, partition recovery, bond no-double-
lock, selective disclosure soundness, canonical determinism, Y_canon
caching, genesis sunset); 7-class threat model (Byzantine peers, network,
timing, compute, hardware, patient, social); 8 composition properties
(C1–C8) at the seams between layers; recommended tooling (TLA+ primary
for safety/liveness; PRISM for the e-process statistical claims; Coq/Lean
as v2.0 deliverable).

The RFC is **specification of intent**, not the TLA+ code. It declares
what the verification must prove; the verification artefact (~6–12 weeks
of TLA+-fluent contributor time) is a downstream engineering deliverable.

**Why:** the reviewer was correct that composition-level failures are the
expensive ones to find late. Spec-of-intent now lets the verification be
commissioned with a clear deliverable. The spec is the cheap part; the
verification work is the cost. Doing them in this order lets the project
attract the right TLA+ contributor rather than guess what the verification
should prove.

### N4 — IMPLEMENTATION.md added at repo root

**Was:** the corpus described the "all-in-one reference client" without
decomposing it. Prospective contributors had no surface to engage with —
"come help build the client" did not specify what "the client" actually
is. The earlier corpus estimate of "6–12 months with 2–4 engineers" was
the reviewer's correctly-identified 3× underestimate.

**Now:** IMPLEMENTATION.md at repo root (not an RFC; project-management
document). Decomposes the client into **15 sub-components** grouped into
6 layers: foundation (crypto/CBOR/TEE), network (transport/DHT/gossip),
reputation+identity (L1 CRDT/L2 chain/identity service), cache
(distributed cache/embedding service), inference (canonical kernels/
orchestration), economy+coordination (ledger/bond accounting). Each
component: spec reference, effort in engineer-months, dependencies,
notes. Critical-path identified (TEE harness → L1 CRDT → L2 chain →
inference orchestration → bond accounting = ~22 EM unbreakable). Six-
phase plan over 36 months for a team of 4.

The honest figure: **72–92 engineer-months total**, **24–36 months
wall-clock with a team of 4–6** including integration testing and
security audit. Communicated explicitly so prospective contributors and
funders see the true cost.

**Why:** the implementation timeline is not a marketing surface — it is
the cost-of-entry contributors need to know honestly before deciding to
participate. Hiding it (or implicitly underestimating it 3×) discourages
serious engagement.

The README has been amended to link IMPLEMENTATION.md from the top
navigation chip line.

### N5 — RFC-0007 promoted from Planned to Draft v0.1

**Was:** GOVERNANCE.md sketched the TSC (7 seats, 2/3 majority, 2-year
staggered terms, transition from BDFL) but did not specify mechanics:
who elects the 3 subsystem maintainers, what qualifies as a "subsystem",
how the chair is elected, conflicts-of-interest, off-cycle removal. The
reviewer correctly noted: "needs to be written before sunset, not after."

**Now:** RFC-0007 promoted to Draft v0.1. Specifies:

- **Seven seats** as 3 subsystem maintainers + 3 contributor-at-large +
  1 chair (transitional 24mo BDFL, then 1-year renewable-once elected).
- **Three subsystems** with explicit RFC boundaries: Verification &
  Numerics (RFC-0003 + RFC-0009 + RFC-0011); Reputation & Identity
  (RFC-0004 + RFC-0005 + RFC-0008 crypto primitives); Cache, Economy &
  Coordination (RFC-0001 + RFC-0002 + RFC-0006 + RFC-0010).
- **Election mechanics**: 14-day nomination + 14-day vote (subsystem
  plurality with run-off; at-large STV); transition election extended to
  60-day nomination + 30-day vote.
- **Eligibility**: voter and candidate eligibility based on merged-work
  in trailing 12 months in the relevant scope.
- **Decision thresholds** (per GOVERNANCE.md): simple majority for
  routine, 2/3 for canonical artefacts (no chair tiebreak), unanimous
  for governance changes. Quorum: 5 of 7 for routine/canonical, all 7
  for governance.
- **Conflicts of interest**: disclosure + recusal mandatory; thresholds
  recompute against remaining present seats; petition-based recusal
  available to the contributor body.
- **Off-cycle removal**: 2/3 of remaining seats + signed petition by
  ≥25% of relevant contributors. Intentionally high bar.
- **Transition flow at v1.0**: BDFL declares → 60-day nomination →
  30-day election → first TSC convenes with outgoing BDFL as chair for
  24 months → BDFL chair role expires, TSC elects new chair from among
  members.
- **Amendment process**: during v0.x, BDFL after 4-week public comment
  (extended from standard); after v1.0, unanimous TSC + 60-day comment
  per GOVERNANCE.md.

**Why:** governance mechanics that people will need to rely on at v1.0
must be specified before they need to rely on them. Drafting RFC-0007 in
v0.x gives the contributor body time to argue with the mechanics before
anything is at stake. The reviewer was correct that "forthcoming" after
v1.0 is too late.

### State after Round 2

All HARD items from the multi-persona review are now closed (Round 1
closed the four BLOCKs; Round 2 closes N3, N4, N5). EDGE-tier register
corrections (Round 3) are the only review items remaining. The spec
corpus is now genuinely review-cycle-complete:

- 11 RFCs (0001–0011, with 0007 promoted) + GOVERNANCE.md + MANIFESTO.md
- 2 root-level documents (README.md + IMPLEMENTATION.md)
- 1 CHANGELOG.md tracking every substantive decision

The remaining open items are either Round 3 register corrections,
testnet-empirical (b2), explicit irreducible residuals handed to the
credible-fork-threat, or downstream engineering work (the TLA+ artefact,
the reference client itself).

---
