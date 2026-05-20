# RFC-0009 — Canonical Numerics for Verifiable Inference

**Status:** Draft · v0.1 (early — substantial revision expected once the reference kernel library exists and the b2 testnet measures its honest-noise floor)
**Topic:** The numerical recipe that two honest peers, on different hardware, must agree on bit-for-bit when committing to verifiable inference.
**Audience:** You, if you have ever stared at two correct-looking GPU implementations producing slightly different logits and wanted to scream.
**Depends on:** [RFC-0003](./RFC-0003-verification.md), [RFC-0008](./RFC-0008-wire-formats.md)

---

## What this RFC tries to do

RFC-0003 §"Canonical numerics on the audited path" names a requirement and a
design lever. The requirement is that two honest peers, running the same
canonical recipe on the same model with the same input, produce bit-identical
(or at least tolerance-bound-identical) logits. The lever is that mandating
this recipe on the audited path shrinks the honest-noise std `σ`, raises the
effect size `e = Δ/σ`, and lowers the viability threshold `T ∝ √κ`.

The requirement is not satisfied by current production LLM inference stacks.
Different GPU vendors, different attention kernel implementations, different
cuBLAS versions, different floating-point reduction orders, different
quantization schemes — all produce slightly different logits for the same
input on the same model. The discrepancies are *small* in absolute terms but
*not zero*, and they sit precisely in the dangerous band where they overlap
with the discrepancy a quantization fraud (FP8 declared as BF16) would
introduce.

This document specifies the canonical recipe. It is the hardest deliverable
in the corpus, and we say so plainly. It is genuinely partial: a starting
recipe that the testnet will refine.

---

## Why this is hard

Three properties of modern LLM inference make bit-exact reproduction
non-trivial:

**Floating-point arithmetic is not associative.** `(a + b) + c` does not always
equal `a + (b + c)` in IEEE-754 floating-point — the rounding occurs at
different points and produces different results. A matrix multiplication is a
massive accumulation of such operations. Different GPUs (and different kernels
on the same GPU) may choose different summation orders for performance.

**Reduction order is implementation-defined.** Attention computation involves
reductions over sequence length. FlashAttention reorders these reductions for
memory efficiency; SDPA (PyTorch's scaled-dot-product-attention) reorders
differently; naive attention reorders differently again. None is "wrong";
they are all correct implementations of the same mathematical operation.

**Mixed-precision arithmetic compounds these effects.** Models served in BF16
or FP16 accumulate intermediate results in FP32, then downcast. Different
implementations may downcast at different points. The differences are tiny
per operation and large in aggregate over an 80-layer transformer.

The result: two correct implementations of "the same model" on different
hardware will produce, for the same input, logits that differ by 10⁻³ to 10⁻¹
in absolute value at the unembedding output. The top-1 token usually agrees;
the top-5 ranking sometimes disagrees; the next-token probability
distribution always differs slightly.

None of this is fraud. It is the price of floating-point arithmetic on
heterogeneous hardware. RFC-0003's verification, naively applied, cannot tell
this noise from quantization fraud — both are "small perturbations of the
correct computation" by the same magnitude. That is why a canonical recipe is
needed.

---

## The recipe — INT8 integer-domain inference on the audited path

The recipe rests on one observation: **integer arithmetic is associative**.
If we lift the audited path out of floating-point and into integers,
reduction order ceases to matter. Two honest implementations of the same
integer arithmetic produce bit-identical outputs regardless of hardware.

### 1 · INT8 weight quantization with a fixed scheme

The audited path runs the model with **INT8 weights** under the **GPTQ
quantization scheme** with per-channel symmetric scaling, group size 128.
This is a well-studied quantization regime: per-channel scales are computed
deterministically from the FP16 reference weights, and the resulting INT8
quantization preserves perplexity within ~0.1–0.3 of the FP16 reference for
all major open-weight LLM families tested (Llama, Mistral, Qwen, DeepSeek,
Phi).

The quantized weights are *not* the weights the producer serves to the end
user. The producer serves whatever it prefers (FP16, FP8, INT8 — its choice,
its hardware). The producer *additionally* commits, on the audited path, to
the output that the canonical INT8-GPTQ-128 quantized model produces.

### 2 · Integer-domain activations on the audited path

Activations on the audited path are quantized at every layer boundary to
INT8, using per-token symmetric dynamic quantization with the scale carried
in FP32. Matrix multiplications are performed as INT8 × INT8 → INT32
accumulation, then dequantized to FP32, then rescaled and re-quantized for
the next layer.

Why this works: the INT8 × INT8 → INT32 product is exact (no rounding loss);
the only floating-point rounding occurs at the dequantize/rescale step, which
is deterministic given the scale values, which are themselves derived
deterministically from the input.

### 3 · The fixed accumulation order

For the INT8 × INT8 → INT32 matrix multiply, the canonical recipe pins:
- **Row-major iteration order** for the output matrix.
- **Inner-product summation** in a left-to-right reduction over the inner
  dimension.
- **No tiling-induced reordering**; this constrains GPU kernel choice and is
  the source of the verifiability tax (~2–4× slowdown vs the best-tuned GPU
  attention kernels).

For the attention computation specifically:
- Softmax is computed in **FP32 with a fixed numerical-stability
  subtraction** (subtract the max along the reduction axis before
  exponentiation, then renormalise).
- The subsequent matmul follows the standard recipe.

### 4 · Deterministic flags

Where a runtime supports them, the canonical recipe additionally requires:
- `CUDA_LAUNCH_BLOCKING=1` (on NVIDIA) for deterministic kernel scheduling.
- cuBLAS deterministic mode (`CUBLAS_PEDANTIC_MATH`) when calling cuBLAS.
- Equivalents on AMD ROCm (`HIP_LAUNCH_BLOCKING=1`,
  `--amdgpu-precise-fp32-mul`).
- On ARM (NEON/SVE): scalar reductions, no vectorised reductions for the
  audited path.
- On CPU: single-threaded execution for the audited path (no OMP, no Rayon),
  or thread-count fixed and reduction-order fixed via the reference kernel
  library.

### 5 · The unembedding step

The final unembedding projection (hidden-state to logits over vocabulary) is
performed in **FP32**, also in the fixed canonical reduction order. The
output logits are returned as FP32 values, then quantized to a fixed-point
representation (q24 — 24 fractional bits) for hashing and Merkle-tree
inclusion.

The q24 representation gives 7 decimal digits of precision, well above what
any honest discrepancy ever produces, while ensuring two implementations that
agree to 7 digits produce bit-identical hashable values.

---

## Why this works (when it works)

Under the recipe above, the audited-path computation is **deterministic up to
the dequantize/rescale rounding**, which is itself deterministic given
inputs. Two honest implementations on different hardware (Intel x86, ARM,
Apple Silicon, NVIDIA, AMD) produce bit-identical INT32 accumulator outputs;
they produce bit-identical FP32 dequantized outputs given the same FP16
scales; they produce bit-identical q24 logits.

The honest-noise std `σ` on the audited path, under this recipe, is expected
to be **essentially zero** for the canonical INT8-GPTQ-128 model — not "small
but tolerable," but actually identical bytes.

A peer that fails to produce the canonical output for a known-honest input
is either implementing the recipe wrong, or running compromised code. Either
way, slashable.

---

## Why this is partial

Honest caveats, in order of severity:

### The recipe is not implemented yet

The reference kernel library — call it `ants-canonical-kernels`, to be
written in Rust with hand-tuned SIMD (AVX2/AVX-512 for x86, NEON for ARM,
Metal Performance Shaders shim for Apple Silicon, ROCm shim for AMD, CUDA
shim for NVIDIA) — does not exist. It is the second-largest piece of
implementation work in the corpus (after the all-in-one reference client),
estimated 6–12 months for 1–2 engineers full-time.

Until it exists, "canonical numerics" is words. We say so.

### Not every model has a validated INT8-GPTQ-128 quantization

GPTQ is well-validated for the major open-weight families (Llama 2/3.x,
Mistral, Qwen 2/3, DeepSeek V2/V3, Phi 3/4, Gemma 2). It is less validated
for newer architectures (mixture-of-experts at extreme scale; state-space
models like Mamba; non-transformer architectures) and for models trained
with QAT (quantization-aware training) that have their own preferred scheme.

For models where GPTQ produces unacceptable perplexity degradation (>1.0
absolute), the recipe falls back to **AWQ** (Activation-aware Weight
Quantization) per-group, with the same per-channel scaling pattern. Where
neither works, the recipe falls back to FP16-with-constrained-kernels (see
below).

### The FP16 fallback is weaker

For models where integer quantization is not viable, the audited path is
**FP16 with the constrained-kernel-set discipline**: single specified
kernel implementation per operation, single specified reduction order,
deterministic flags. This achieves a much smaller honest-noise floor than
"whatever kernel the user picked," but not bit-exact zero — typical
discrepancy of 10⁻⁵ to 10⁻⁴ in logit space, which is detectable by the
e-process aggregate test but not zero.

Under the FP16 fallback, the test in RFC-0003 §Tier 2 still works (the
e-process is robust to a non-zero honest floor; it just needs more
challenged positions for the same statistical power). But the effect-size
`e` and the threshold `T` are different from the integer-recipe case, and
b2 must measure them separately.

### Producer compute overhead

The audited path is *slower* than the producer's preferred fast path. INT8
inference with constrained kernels is typically 2–4× slower than
well-optimised FP16 inference on the same hardware (because the constrained
recipe forbids the tiling and parallelisation tricks that make modern GPU
attention fast).

This is the **verifiability tax** RFC-0003 names. A producer that wants to
serve a query and *also* commit to it must do roughly twice the compute: the
fast path for the user's experience, the audited path for the commitment.
The user sees no slowdown; the producer's NCS earnings per CPU-second on
audited work are roughly halved.

The tax is bounded by the audit-sampling rate. At p=3% Tier 1 sampling,
~3% of inferences pay the tax. At higher tiers (Tier 2 cross-check, Tier 3
triangulation), the tax compounds.

### Side-channel leakage from the audited path

A subtle concern: the audited path may leak information about the input that
the fast-path serving does not (because the canonical recipe has a different
memory-access pattern). Whether this matters for privacy-sensitive queries
needs analysis before any production use of canonical numerics in
TEE-confidential modes.

---

## The reference kernel library

The specification of the kernel library, in skeleton form:

```
ants-canonical-kernels (a Rust crate)
│
├── matmul/
│   ├── int8_int8_int32.rs      // INT8 × INT8 → INT32 GEMM
│   ├── fp32_unembed.rs         // FP32 unembedding (canonical reduction)
│   └── tests/                  // bit-exact test vectors against q24 outputs
│
├── attention/
│   ├── canonical.rs            // softmax + matmul, fixed reduction order
│   └── tests/
│
├── quantize/
│   ├── gptq_per_channel.rs     // GPTQ INT8 quantization from FP16 weights
│   ├── awq_per_group.rs        // AWQ fallback
│   ├── activation_dynamic.rs   // per-token symmetric INT8 activations
│   └── tests/
│
├── fixed_point/
│   ├── q24.rs                  // FP32 ↔ q24 conversion, deterministic
│   └── tests/
│
└── platforms/
    ├── x86_avx2.rs             // AVX2 SIMD implementations
    ├── x86_avx512.rs           // AVX-512 SIMD
    ├── arm_neon.rs             // NEON SIMD
    ├── cpu_scalar.rs           // pure scalar fallback (slow but reproducible)
    └── README.md               // GPU platform shim status
```

GPU platform support is *aspirational* in v0.1. The pragmatic v1.0 target is:
- **CPU canonical implementation works on all reference platforms.** This is
  the foundation; nothing ships without it.
- **GPU acceleration of the canonical path** is added per-platform as time
  allows, with bit-exact validation against the CPU reference for each
  platform-and-kernel combination.

A peer that lacks a GPU canonical kernel for its platform falls back to the
CPU canonical path for the audited-path computation, accepting the
performance penalty. This is acceptable because the audited path is a
*small fraction* of total inference (only the p=3% sampled portion of Tier
1, plus per-position teacher-forced steps for Tier 2 audits, plus the rare
Tier 3 committee work).

---

## Test vectors

Each canonical recipe produces deterministic outputs. The test-vector pack
(stored in `Ants-Community/ants-test-vectors` per RFC-0008 §8) will include,
per supported model checkpoint:

- A small input (10–30 tokens) and its expected canonical-recipe logits, q24,
  for every position.
- The corresponding Merkle root of the per-position logit leaves.
- The expected commit-at-send object's BLAKE3 hash.

A peer that produces a different Merkle root for the same input on the same
canonical-quantized checkpoint is non-conformant. This is the cleanest
possible conformance test for the audited path.

The vectors are generated by the reference kernel library when it ships;
until then, this RFC declares the structure.

---

## Tolerance bounds and the e-process

Even under the INT8 canonical recipe, the *honest* `σ` is not literally zero
— it is the q24 quantization error (~10⁻⁷ absolute), which is below the
detection floor of any plausible verification scheme. In effect: `σ ≈ 0`,
the effect size `e = Δ/σ → ∞` for any non-zero quantization fraud, and the
threshold `T → 0`. RFC-0003's Tier 2 verification of integer-canonical
audited paths is therefore *trivial* in the statistical sense — any
discrepancy is a slash.

Under the FP16 fallback recipe, `σ ≈ 10⁻⁵` to `10⁻⁴`. The e-process test
applies as RFC-0003 specifies; b2 will measure the actual `σ` and the
corresponding `e` for known fraud classes (FP8-as-FP16, smaller-model
substitution, etc.) and the table of (model, regime) → {Tier 2 GO with
INT8 | Tier 2 GO with FP16 | Tier 3 fallback} is filled in.

---

## What we have not figured out yet

This RFC is the most honestly-incomplete one in the corpus. Open problems:

- **Quantization-aware-trained (QAT) models** with their own preferred
  schemes (e.g. some Mistral and Qwen variants). The recipe needs a path
  for these without forcing re-training.
- **Mixture-of-experts** (MoE) models route different tokens to different
  experts. The canonical recipe needs a deterministic routing policy that
  doesn't depend on (e.g.) which experts a tile-scheduler happens to assign.
  This is non-trivial.
- **Speculative decoding** breaks the "one forward pass per token" assumption
  that the audited path rests on. A producer using speculative decoding for
  serving must still produce the canonical audited-path output, which means
  re-running without speculation for the commitment — paying double tax.
- **KV-cache reuse** across queries. The audited-path teacher-forced
  per-position recompute (RFC-0003 Tier 2) needs the prefix's KV state. If
  the auditor recomputes it from scratch every time, audit cost scales
  with prefix length. KV-cache reuse during audits is desirable but
  requires the cache itself to be canonical.
- **GPU canonical kernels for NVIDIA Hopper / Blackwell** are an enormous
  engineering project that depends on CUDA's PTX-level determinism guarantees,
  which are imperfect. Expect months of work per GPU generation.
- **Apple Silicon canonical kernels** must work with Metal, which gives the
  developer less control over reduction order than CUDA does. Whether
  bit-exactness is achievable on MPS is an open empirical question.
- **The verifiability-tax economics.** If the tax is 2× compute on audited
  inferences, and the audit rate is p=3%, the network's effective compute
  cost is +6% above the no-verification baseline. This is borne by the
  producers, not the network. Whether it remains rational to produce under
  this cost — especially for community-layer barter peers — needs
  modelling. If producers reject the tax, audited Tier 2 verification stops
  happening voluntarily.
- **Closed-weight models** (Claude, GPT-class served via RFC-0006 commercial
  bridges) cannot be canonically re-run by an independent auditor; they live
  outside the integer-recipe regime. Tier 3 triangulation against
  *different* open-weight models is the v0.1 answer, but it is a much
  weaker check than canonical re-execution. Open.

---

## How to contribute

Open an issue on `Ants-Community/ants` with the tag `rfc-0009`. The two kinds
of contribution most valuable right now:

- **Empirical evidence on quantization quality.** Benchmarks of GPTQ
  INT8-per-channel-symmetric on specific model families, with perplexity
  measurements vs the FP16 reference. The current "0.1–0.3 perplexity
  degradation" claim is informed but not exhaustively measured.
- **GPU kernel implementations** of the canonical recipe, with bit-exact
  test-vector conformance. This is the hardest engineering work the corpus
  asks for and is where the project benefits most from open contribution.

Routine clarification PRs ("the wording in §3 is ambiguous") are merged
quickly. Substantive proposals to change the recipe (e.g. "AWQ should be
the default, not GPTQ") go through the full amendment process with
empirical justification.

The document is CC0. Adopt the recipe for your own verifiable-inference
protocol if it helps.

---

*Drafted by the ANTS founding contributors, May 2026.
The integers do not lie. The floats sometimes do.*
