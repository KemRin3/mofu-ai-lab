# ZONOS2 AVX-512 compatibility investigation

## Scope

Reproduction note for a possible CPU instruction-set compatibility issue in the Windows distribution of `Zyphra/zonos2.cpp` v0.5.1. The goal is to preserve a concise, public report that can support an upstream issue if a controlled rebuild resolves the problem.

## Recorded observations

Read-only review of existing local files on 2026-09-05 found the evidence below. No executables, experiments, or builds were run during this review. Paths below are relative to the local installation or source checkout; raw local logs are not included in this repository.

- `README_LOCAL.md` identifies the upstream as `Zyphra/zonos2.cpp`, release v0.5.1, and the original distribution as the official Windows x64 Vulkan portable ZIP. It records that the active executables were subsequently replaced with a local rebuild. The ZIP and original executable backup are present; release provenance was not independently authenticated in this review.
- The same local note records Ryzen 7 5700X hardware, lack of AVX-512 support, Q6_K usage, and successful Japanese TTS tests.
- It records `0xC000001D` failures in the original speaker-encoder paths for both `spk-encoder-cli.exe` and `zonos2-server.exe`, with offsets `0xF635` and `0x82205`, respectively. It describes bytes beginning `62 F1 7C 48 10 ...` at those positions as AVX-512 EVEX/ZMM instructions. This review confirms that the investigation record exists; no fresh disassembly or crash reproduction was performed.
- `models/spk-encoder.gguf` is present (24,299,552 bytes). `README_LOCAL.md` reports that official model SHA-256 values match the distribution values, but does not provide the speaker-encoder digest pair needed to independently check that claim here.
- `build-avx2-vulkan/CMakeCache.txt` in the local source checkout contains `GGML_NATIVE=OFF`, `GGML_AVX2=ON`, `GGML_VULKAN=ON`, and `GGML_AVX512`, `GGML_AVX512_BF16`, `GGML_AVX512_VBMI`, and `GGML_AVX512_VNNI` all set to `OFF`. It also specifies a Vulkan SDK shader compiler.
- Existing `outputs/server-avx2.stderr.log` and `outputs/start-zonos2-avx2.stderr.log` show the RTX 5060 Ti Vulkan backend, backbone loading, and successful speaker-encoder loading (`spk: loaded 78 tensors`, `enc_dim=2048`).
- `README_LOCAL.md` records successful embedding generation and Japanese Voice Clone output after the rebuild. The reviewed server log supports successful encoder loading, but is not a complete command-by-command record of those tests.

## Previous investigation notes

- The exact `spk-encoder-cli.exe --clone` and `spk-encoder-cli.exe --raw` failure invocations come from previous investigation notes. Their command-specific reproduction records were not verified in the files reviewed this time.
- The speaker-encoder SHA-256 match remains a previously reported result, not an independently verified digest comparison in this review.

## What is not confirmed

The root cause has not been established. In particular, the presence of AVX-512 instructions near the failure does not by itself prove that they were executed on the crashing path or that they are the only defect.

## Working hypothesis

The Windows-distributed binary may contain AVX-512 instructions through build-time native CPU optimization or a related compiler/build configuration. On a Ryzen 7 5700X, this could produce the observed illegal-instruction exception when the speaker encoder is used.

This is a compatibility hypothesis, not a confirmed diagnosis.

## Rebuild configuration and follow-up

The local setup note reports a completed v0.5.1 rebuild, and the existing CMake cache supports the following configuration:

- `GGML_NATIVE=OFF`
- AVX-512 disabled
- Vulkan enabled

For an upstream report, preserve a comparison of the rebuilt executable and official Windows binary using the same model files and prompts, with exact build flags, compiler/toolchain version, GPU driver context, commands, exit codes, and outcomes for all three paths. This comparison was not run during the read-only review.

## Current status

The earlier Vulkan SDK blocker is superseded by the existing setup record and CMake cache: the local record reports that the rebuild was completed and fixed the affected functionality, and existing server logs show encoder loading succeeds. This supports the compatibility hypothesis but does not independently establish which build setting introduced the original failing instructions. The root cause is not declared confirmed.

## Upstream issue readiness

To substantiate the recorded rebuild result, an upstream report should include:

1. the three minimal reproduction commands;
2. the official-binary exception code and affected CPU model;
3. the speaker-encoder model checksum match;
4. the rebuilt version's exact configuration and outcome; and
5. a request to review Windows release build CPU-ISA settings.
