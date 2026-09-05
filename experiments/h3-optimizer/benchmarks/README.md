# H3 Optimizer benchmark records

All results below are historical or saved measurements reviewed on 2026-09-05. No new measurements were made. Source IDs are defined in the [research note](../README.md). A dash means not established in the reviewed record, not zero. Compare only explicitly paired measurements with matching scope.

## Current operational benchmark — 2026-09-02 (S6)

Maintainer-supplied operational results; no new run was performed for this update. These are separate from the historical acceptance and attention-budget studies below.

| Workflow condition | Value |
| --- | --- |
| Workflow | MiniMax H3 Reference-to-Video |
| Resolution | 544 × 800 |
| Duration / fps / frames | 8.0 seconds / 24 fps / 192 frames |
| Sampling steps | 8 |
| Sampler | `res_multistep` |
| H3 Quality Sigma Controller: steps | 8 |
| H3 Quality Sigma Controller: profile | `turbo_balanced` |
| H3 Quality Sigma Controller: strength | 1.00 |
| H3 Quality Sigma Controller: alpha / beta | 0.60 / 0.60 |
| Model | `minimax_h3_ref2va_pruned_int8_convrot.safetensors` |
| VAE | `minimax_h3_video_vae_int8_convrot.safetensors` |
| CLIP | `qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors` |
| LoRA | `minimax_h3_turbo_v4_step600_ema.safetensors` |
| LoRA strength | 0.7 |
| Adaptive Optimizer | `quality` preset |

Sigma Controller strength 1.0 and LoRA strength 0.7 are separate settings. These current model/VAE/CLIP/LoRA and preset conditions must not be applied to historical QUALITY/FAST or other past runs. Artifact hashes and other unspecified settings must not be borrowed from historical runs.

| Sampling measurement | Seconds |
| --- | ---: |
| Warm-up (excluded from average) | 181.17 |
| Run 1 | 132.65 |
| Run 2 | 132.96 |
| Reported average | 132.80 |

The reported average is retained as supplied; the lossless comparison below records the baseline at higher precision (132.8016 s). Do not recalculate it from rounded run displays.

| VAE decode, current operational conditions | Warm-up s | Run 1 s | Run 2 s | Average s | Decision |
| --- | ---: | ---: | ---: | ---: | --- |
| Standard H3 VAE | 18.79 | 17.92 | 17.99 | 17.95 | Continue using Standard H3 VAE |
| TensorRT H3 VAE | 32.64 | 20.02 | 20.00 | 20.01 | Rejected for operational use: about 11% slower; retained for validation |

Both VAEs decode the same latent, shape `[1, 24, 57, 50, 34]`, with one warm-up followed by two measured runs. Sampling time is excluded. Standard / TensorRT = **0.90×** (ratio of durations); TensorRT is about 11% slower under these operational conditions. Sampling and VAE times are separate measurements; no combined E2E duration is asserted. Output-quality comparison remains unverified in the supplied record.

### Lossless rejected experiments (S6)

| Sampling candidate | Baseline s | Candidate s | Baseline / candidate | Numerical result | Decision |
| --- | ---: | ---: | ---: | --- | --- |
| Remove `v = v.clone()` | 132.8016 | 132.7761 | 1.0002× | Latent bit-exact | Rejected and reverted; difference within measurement noise |
| Aggregate GPU-to-CPU scalar synchronization | 132.8016 | 132.9112 | 0.9992× | Latent bit-exact | Rejected and reverted; no speed benefit |

These operational sampling comparisons do not replace the separate MLP/component tests. Bit-exactness is the reported latent comparison for these tests, not a claim about all workflows.

## Historical generation conditions

| Field | Historical QUALITY / FAST acceptance (S1) | Turbo attention-budget study (S1, with runtime evidence from S4) |
| --- | --- | --- |
| GPU | RTX 5060 Ti / SM120 | RTX 5060 Ti / SM120 |
| Resolution | 544 × 800 | 544 × 800 |
| Frames / fps / duration | 124 / — / — | 192 / 24 / 8 s nominal (192 ÷ 24) |
| Steps | 20 | 8 |
| Model | S1 describes current FL2VA ConvRot INT8; exact acceptance artifact hash not recorded here | Pruned H3 with ConvRot INT8 linear path; exact checkpoint hash not recovered |
| VAE | Exact acceptance variant/settings not recovered | Runtime loads H3 audio/video VAEs; exact artifacts/settings not recovered |
| LoRA | Not established for this acceptance run | S4 records `minimax_h3_turbo_v4_step600_ema.safetensors`, strength 0.7; exact per-band workflow linkage not recovered |
| Sampler | Not established | S4 records native ModelSamplingAV single-schedule Euler, 8 steps |
| Preset | QUALITY or experimental FAST, as labelled below | QUALITY comparator; Turbo FULL and optional layer bands |

S4 is runtime corroboration, not proof that every run in its multi-workload log matches the S1 table. Do not transfer its LoRA/sampler fields to unrelated historical runs without the saved workflow. Filenames above identify artifacts only; no weights are included.

## Historical acceptance and generation timings

| Record | Sampler s | VAE decode s | Total s | Temperature/boundary and interpretation |
| --- | ---: | ---: | ---: | --- |
| QUALITY reference, 20 steps / 124 frames (S1) | 97.819 | — | 134.185 | Source calls total warm-equivalent; exact reconstruction boundary not supplied. P6 11 REAL / 6 CACHE / 3 FORECAST, no Sol fallback. Historical optimized reference. |
| FAST acceptance, same listed dimensions (S1) | 143.138 | — | 218.329 | Cold: includes first-use model/kernel setup. Not comparable as a warmed replacement for QUALITY. |
| Turbo-study QUALITY comparator, 8 steps / 192 frames (S1) | 164 | — | — | Historical sampler result; cold/warm classification not explicitly documented. |
| Turbo FULL, same study (S1) | 159 | — | — | Decoded-video hashes match QUALITY comparator per S1; no broad quality claim. |
| Turbo bands 0 / 1 / 2 / 3 / 4 (S1) | 159 / 157 / 158 / 156 / 149 | — | — | Initialization variance; stable middle/protected-step interval about 17.1–17.4 s. Band 2 balanced is only 0.6% faster than FULL. |

The 20-step / 124-frame QUALITY entry remains a **historical acceptance result**, not a direct speed comparator for the current 8-step / 192-frame benchmark. There is no matched unoptimized E2E baseline or isolated VAE timing in these historical results. Later memory changes were not followed by a new QUALITY E2E measurement. The S4 log ends with an interrupted subsequent workload and reports 387.71 s for that prompt; this is excluded as a successful E2E benchmark.

## Memory and MLP components

S1 measurements dated 2026-08-31; S2 corroborates the method. RTX 5060 Ti / SM120, inference mode, three warmups, ten calls, medians. Actual FL2VA ConvRot weights with synthetic activations. Extra peak means PyTorch live allocation above test start, not reserved memory or whole-GPU VRAM.

| Paired test | CUDA ms before → after | Wall ms before → after | Extra peak MiB before → after | Output evidence / decision |
| --- | ---: | ---: | ---: | --- |
| FinalLayer full → chunk 4096; 15,500 video + 438 audio rows, hidden 5376 | 6.145 → 5.964 | 6.235 → 5.998 | 636.12 → 173.80 | FP32 video relative L2 4.92e-6, cosine approximately 1; audio bit-identical, finite. Adopted only under pressure. |
| FinalLayer full → chunk 8192, same input | 6.145 → 6.163 | 6.235 → 6.196 | 636.12 → 341.80 | Video relative L2 3.29e-6, finite; not selected. |
| Sol QKV early release; 16,194 tokens, 56 heads, actual INT8 ConvRot projections | 69.278 → 69.887 | 69.351 → 69.964 | 1163.02 → 893.26 | BF16 output bit-identical, finite; adopted for memory, not speed. |
| Output projection alone, full → chunk 4096 | 10.430 → 9.740 | 10.513 → 9.779 | 276.81 → 236.07 | Bit-identical; isolated benefit insufficient for adoption. |
| Sol + projection after QKV release, full → chunk 4096 | 71.478 → 70.772 | 71.555 → 70.837 | 893.26 → 893.26 | Bit-identical; not adopted. |
| Full MLP → input-release candidate; BF16 [16194,5376], block-0 checkpoint weights | 51.508 → 51.560 | 51.561 → 51.594 | 1440.166 → 1274.114 | Bit-identical, relative L2 0; rejected. Both arms include identical input clone cost. |

MLP subcomponents were fc1 31.082 ms and fused activation/quantization + fc2 20.060 ms. Input-release saved 166.052 MiB but did not accelerate execution. No REAL-block/sampler/E2E A/B followed. QKV release lowered live allocation at projection entry from 1205.29 to 541.09 MiB; it does not eliminate the original full QKV allocation.

## Precision and isolated native attention

| Study / scope | Reference CUDA ms | Candidate CUDA ms | Numerical evidence and limit |
| --- | ---: | ---: | --- |
| S1 precision component, 16,194 packed tokens | Sol BF16 29.57 | Residual INT8 QK 24.79 | Reported 16.2% faster; BF16 output shape/dtype correct, no NaN/Inf. No numerical error magnitude or quality equivalence established here. Experimental FAST only. |
| S1 native audit, same-run BF16 QKV [1,8,16194,128], seed 0 | Sol tau 1.3: 4.599 | Native KV100%: 8.610; KV15%: 3.853 | KV100% relative L2 0.01607 vs dense BF16 despite full connectivity. Both candidates finite with correct shape/dtype. No video quality test or integration. |

Native wall times: Sol 4.632 ms, KV100% 8.643 ms, KV15% 3.888 ms. Extra peaks: 33.01 / 82.14 / 88.14 MiB respectively. Native timings include carrier allocation/quantization, routing, and execution. KV15% is about 16.2% faster than its same-run Sol reference, not evidence for a matching sampler gain. The independent router does not reproduce the upstream Python routing/schedule. The precision test and 8-head native audit are different workloads; their Sol times must not be compared.

## MoBA3D saved measurements

S3: PyTorch 2.12.0+cu130, BF16 QKV [1,8,16194,128], video lattice [31,25,20], 694 prefix tokens, three warmups and ten measurements. Times below are **wall milliseconds**, unlike CUDA columns above. Separate primary and Triton runs have separate Sol references.

| Run / backend | First-call wall ms | Warm median wall ms | Interpretation |
| --- | ---: | ---: | --- |
| Primary Sol | 171.870 | 3.587 | Same-run reference |
| Primary native dense | 77.916 | 23.065 | Dense comparison backend; distinct from the native audit above |
| Primary Flex control, block [1,8,16], KV100% | 1847.267 | 114.357 | 32,512 padded tokens; finite BF16 output |
| Primary Flex default [1,8,16], KV25% | — | 37.926 | Flex executor accounts for about 79.6% of reported warm total |
| Primary fastest Flex [4,4,8], KV25% | — | 20.888 | About 5.82× slower than primary Sol |
| Triton-run Sol | 166.245 | 3.588 | Same-run reference |
| Triton-run Flex [4,4,8], KV25% | 1855.644 | 21.022 | 22,272 padded tokens |
| Triton proof of concept, all-in | 943.656 | 20.754 | About 5.78× slower than its Sol reference; saved decision: stop |

Primary KV100% control versus native dense: cosine 0.99999958, relative L2 0.00096975, max absolute difference 0.00024414. Versus Sol: cosine 0.63213837, relative L2 1.07245409. Different reference paths are not interchangeable numerical ground truths. Triton reports valid shape/dtype and finite output but does not establish video quality. No claim is made that these experiments reproduce MiniMax's official production Native Sparse configuration.

## Gaps before a reproducible public comparison

Recover complete workflows, artifact hashes, Optimizer revision, seed/conditioning, VAE settings, and exact timing boundaries for the generation records. S6 supplies the previously missing TensorRT VAE, clone-removal, and scalar-aggregation results and rejection decisions; underlying run artifacts would strengthen reproducibility. The historical v1.1 scalar-cleanup description does not override the later rejected-and-reverted operational experiment. Preserve first-call and warmed results separately and exclude interrupted or competing-GPU runs.
