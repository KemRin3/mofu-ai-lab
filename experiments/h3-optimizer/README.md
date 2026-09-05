# MiniMax H3 Adaptive Optimizer research notes

This research explores faster local MiniMax H3 inference on an RTX 5060 Ti / SM120, prioritizing preservation of observed output quality. This is a synthesis of existing records reviewed on 2026-09-05, not a new benchmark run. Optimizer source, model weights, binaries, generated media, and raw logs are not distributed here.

The sequence is baseline → hypothesis → experiment → measurement → accept or reject. Numerical results and their measurement boundaries are collected in [the benchmark tables](benchmarks/README.md).

## Evidence and baseline

Source identifiers used in both documents:

- **S1:** installed Adaptive Optimizer `README.md`, headed v1.3.0. Its historical QUALITY/FAST results and dated 2026-08-31 memory/MLP investigations provide most acceptance decisions. These are recorded results, not independently repeated measurements.
- **S2:** `tests/benchmark_memory_parts.py` and `tests/benchmark_mlp_quality.py` in that custom node. The measurement helper uses three warmups, ten measured calls, CUDA events, synchronization, and medians. The memory script loads the FL2VA pruned INT8 ConvRot checkpoint.
- **S3:** saved `minimax_h3_moba3d_lab_last.json`, including the FlexAttention comparisons and appended Triton proof of concept. It records PyTorch 2.12.0+cu130, RTX 5060 Ti, and capability 12.0.
- **S4:** saved `h3_turbo_attention_budget_final_stdout.log` and `h3_turbo_attention_budget_final_stderr.log`, plus targeted inspection of the earlier Turbo/race logs. These establish runtime choices and also contain an interrupted workload; not every elapsed number is a valid completed E2E result.
- **S5:** local `h3_native_audit/README.md`, describing the isolated native library and its interface. This is supporting provenance documentation, not the source of the measured speedup; the measurements are in S1.
- **S6:** maintainer-supplied status dated 2026-09-02, added during pre-publication supplementation: current operational sampler/VAE measurements and two rejected lossless experiments. These supplied results were not rerun during this review.

These identifiers refer to local evidence, not downloadable artifacts in this repository. Their machine-specific paths and raw contents have deliberately not been reproduced. The inspected Git history resolves to the parent ComfyUI checkout, so its commit must not be presented as a pinned Optimizer revision.

The historical QUALITY acceptance result is 544 × 800, 124 frames, 20 steps: sampler 97.819 s and a source-labelled **warm-equivalent total** of 134.185 s. It is an optimized reference, not a measured unoptimized baseline. That historical record does not establish a matched optimizer-off E2E comparison or a standalone VAE decode duration. Do not attribute the gap between sampler and total entirely to decode.

## Current operational benchmark — 2026-09-02

S6 records 544 × 800, 8 seconds, 24 fps, 192 frames, and 8 sampling steps with `res_multistep`. The H3 Quality Sigma Controller uses steps **8**, profile **`turbo_balanced`**, strength **1.00**, alpha **0.60**, and beta **0.60**. These are sigma-controller settings; the profile name alone does not identify the Optimizer preset or LoRA strength.

Sampling warm-up was 181.17 s; measured runs were 132.65 s and 132.96 s, with a reported average of **132.80 s** excluding warm-up. Standard H3 VAE averaged **17.95 s**, versus TensorRT H3 VAE **20.01 s** (Standard / TensorRT = **0.90×**). TensorRT was about 11% slower under these operational conditions and was rejected. These are separate sampling and decode measurements, not a measured E2E total.

Keep this current 8-step / 192-frame benchmark separate from the historical 20-step / 124-frame QUALITY acceptance result. Their times must not be used for a direct speed comparison.

## Adopted paths and what is actually demonstrated

| Path | Hypothesis and implementation | Evidence and adoption boundary |
| --- | --- | --- |
| Sol attention | Use the validated SM120 attention backend | QUALITY AUTO prefers Sol, then Direct Sage, then native. The historical QUALITY run reports no Sol fallback. This establishes use, not an isolated E2E gain from Sol alone. |
| Cross-Step REAL/CACHE/FORECAST | Avoid a full transformer evaluation on selected steps using residual reuse or bounded forecast | Historical QUALITY records 11 REAL / 6 CACHE / 3 FORECAST. Early/late retention and change guards remain. This is an approximation with workload-specific acceptance, not proof of universal quality equivalence. Reference-audio QUALITY has stricter gates and at most one FORECAST according to S1. |
| Fused Modulation | Reduce modulation/residual overhead while preserving the tested arithmetic | S1 describes bit-exact BF16 fusion; S4 reports it active without fusion fallback. No isolated timing gain was found in the reviewed records. |
| Memory Engine | Retain full QKV/MLP when they fit; bound work under pressure | Adopted adaptive policy. QKV early release saves measured live allocation with bit-identical outputs, but is not a demonstrated speedup. FinalLayer chunking is pressure-only, with small floating-point differences; ordinary QUALITY remains full. |
| Token Engine | Avoid unnecessary selection/layout work when planned retention is 100%; optionally prune elsewhere | QUALITY uses the full-token direct path on critical REAL steps. FAST aggressive pruning remains experimental. Enabled policy is not evidence of a separately measured quality-safe acceleration. |
| Precision | Keep QUALITY BF16; evaluate residual INT8 QK separately | BF16 remains the quality reference. Residual INT8 QK is faster in a component test, but selected for experimental FAST/compatible SM120 only. Existing checkpoint-native ConvRot quantized linear execution is retained. |

S1 historically describes v1.1 scalar synchronization cleanup as implemented. The later S6 operational experiment records GPU-to-CPU scalar aggregation as rejected and reverted, as detailed below; the earlier description must not be used as evidence that the later candidate remains adopted.

Turbo is a separate 8-step regime. Its `TURBO_BALANCED` FULL default disables token pruning, forces REAL, and uses BF16 Sol. LoRA and sampler are supplied separately. This must not be confused with 20-step QUALITY or FAST.

## Rejected, stopped, and experimental candidates

| Candidate | Hypothesis → test | Measurement and numerical/quality evidence | Decision |
| --- | --- | --- | --- |
| MLP input release | Free normalized input after fc1 while retaining existing fused activation/quantization and fc2 | CUDA 51.508 → 51.560 ms; extra live allocation 1440.166 → 1274.114 MiB. Bit-identical BF16 output, relative L2 0. | Rejected: no speed gain; 166.052 MiB saving below the recorded several-hundred-MiB adoption target. Stop condition below 2% timing difference reached; no sampler/E2E follow-up. |
| Output projection chunk 4096 | Reduce projection cost and memory by splitting rows | Projection alone improves, but whole Sol+projection is 71.478 → 70.772 ms and peak remains 893.26 MiB after QKV release. Bit-identical outputs. | Not adopted: insufficient whole-path benefit; extra output buffer/launches. |
| FinalLayer chunk 8192 | Explore a larger chunk | CUDA 6.145 → 6.163 ms; extra peak 636.12 → 341.80 MiB. Video relative L2 3.29e-6. | Not selected; 4096 provides a better pressure-relief option in the recorded test. |
| Turbo attention budgets | Raise Sol tau in isolated layer bands to reduce attention cost | FULL 159 s; bands 159/157/158/156/149 s. Protected/middle intervals remain approximately 17.1–17.4 s. S1 attributes the lowest total to initialization variation; balanced improves only 0.6%. | Reduced budget not adopted as default. FULL remains default; band candidates are opt-in experiments. Only QUALITY vs FULL has a reported decoded-video hash match. |
| MoBA3D FlexAttention | Route 3D video blocks to fewer KV blocks while protecting conditioning | Default layout pads 16,194 to 32,512 tokens. Fastest primary 25% candidate: 20.888 ms wall vs same-run Sol 3.587 ms. Dense-control numerical comparison is in the benchmark table. | Research-only; slower on this workload. This is not a reproduction of MiniMax's production Native Sparse configuration. |
| MoBA3D Triton executor | Replace the dominant sparse executor | All-in warm wall 20.754 ms vs same-run Sol 3.588 ms; correct shape/dtype and finite output. | Saved decision: stop. Finite output alone is not a video-quality validation. |
| Native Kitchen sparse | Quantized native traversal with an independently written router | KV15%: 3.853 ms CUDA vs same-run Sol 4.599 ms. KV100% control still has relative L2 0.01607 vs dense BF16. | Isolated quality-risk candidate, not integrated; no video quality test. Not a benchmark of the upstream Python router or its early-step schedule. |
| INT8 P·V | Reduce attention arithmetic cost further | S1 reports inconsistent speed advantage and higher numerical error; no numeric pair recovered here. | Not selected. Do not invent a speedup or an error magnitude. |
| TensorRT VAE (S6) | Accelerate decode under current operational conditions | Standard 17.95 s average vs TensorRT 20.01 s; Standard / TensorRT = 0.90×. No output-quality comparison supplied here. | Rejected: about 11% slower in the current practical workload. |
| Removal of `v = v.clone()` (S6) | Remove a copy without changing latent output | Sampling baseline 132.8016 s → candidate 132.7761 s; 1.0002×; latent bit-exact. | Rejected and reverted: speed difference within measurement noise. |
| GPU-to-CPU scalar synchronization aggregation (S6) | Batch scalar transfers without changing latent output | Sampling baseline 132.8016 s → candidate 132.9112 s; 0.9992×; latent bit-exact. | Rejected and reverted: no speed benefit. |

Earlier MLP runs with a competing model server or instrumentation retaining tensors are explicitly invalidated in S1 and excluded from the tables.

## Remaining research and reproduction limits

The historical profile in S1 classifies MLP at 2575 ms and Sol at 2178 ms; it is not a fresh Turbo-step attribution. MLP remains a target, but the tested input-release candidate did not justify adoption. Streamed Q is blocked by the current Sol contract: equal Q/K/V lengths, no external query offset, and routing based on global query rows. An adapter alone is not a demonstrated solution. Embedding lifetime remains a memory-only candidate with wrapper compatibility constraints. Sparse/precision work still needs video-level quality assessment before normal adoption.

Before a fresh comparison, preserve exact model/VAE/LoRA identifiers and hashes, prompt and seed, conditioning, resolution, frames/fps, steps, sampler/sigma schedule, preset and overrides, runtime versions, and active fallback reports. Missing historical fields are listed in the benchmark document rather than inferred.

Use the same warmup and measurement boundary in both arms. Isolate competing GPU work, distinguish first-use compilation/model loading from warmed execution, and do not equate PyTorch live allocation with reserved or whole-device memory. Component tests with synthetic activations and actual weights do not establish a sampler or E2E speedup. Compare output shape/dtype/finiteness and numerical error, then evaluate generated video/audio quality separately.

The existing source-tree benchmark entry points include `tests/benchmark_memory_parts.py` (FinalLayer; `--sol` for QKV lifetime, `--output` for projection, `--sol --bounded` for combined projection), `tests/benchmark_mlp_quality.py`, and `tests/benchmark_native_kitchen.py`. They are evidence references, not bundled runnable reproductions; model/runtime paths require local configuration. No tests or inference were executed for this publication review.
