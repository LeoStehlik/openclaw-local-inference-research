# Tiercel slot-checkpoint backport results

Date: 2026-10-30 UTC

## Verdict

PASS for Qwen3.8-27B Q4 + Q4 DFlash2 checkpoint persistence, both within one server process and after a full server restart.

The installed stock runtime returns a successful slot restore but does not restore the recurrent prompt checkpoints. The backported upstream fix restores those checkpoints and reproduces the same prefix reuse as the live in-memory path.

The candidate was tested only in a disposable localhost process. The installed production runtime was not replaced and the original service was restored healthy after the test.

## Source

- Local clone: `/home/less/src/llama.cpp-tiercel`
- Leo fork branch: <https://github.com/LeoStehlik/llama.cpp/tree/fix/slot-checkpoint-persistence>
- Base: `Nathanw1014/llama.cpp:strix-halo-vulkan` at `629d2f7b45285e3f7556648c50c167159c532476`
- Final tested head: `a82001bd382ea54481603be113ced229a21db8d2`
- Upstream backport: `033df86b69ec1a333eb241f0c16325a4d43dcff5`, llama.cpp PR #26004
- Fork-policy commit: `23f0039d02e8cfa8e042d1b3419f7702a315bab7`
- Backport commit: `a82001bd382ea54481603be113ced229a21db8d2`

The backport retains the upstream cherry-pick reference and changes the same six source/test files as upstream. The only semantic conflict adaptations combine upstream's new successful-load assertions with the Strix Halo branch's existing on-device flags for rolling speculative checkpoints:

```cpp
GGML_ASSERT(ckpt.load_*(..., LLAMA_STATE_SEQ_FLAGS_PARTIAL_ONLY | LLAMA_STATE_SEQ_FLAGS_ON_DEVICE));
```

Regular prompt checkpoints restored from a slot file remain host-resident, as in upstream.

## Build

- Candidate identity: build 11003, commit `a82001bd3`
- Compiler: GCC/G++ 14.3.0
- Shader compiler: exact production pin `shaderc v2026.3-dev`, commit `49a8724d561c13db22b52f99f2a0e2707a9a9e3c`
- CMake features: Vulkan, `GGML_NATIVE=ON`, Trellis, EXL3, `LLAMA_CURL=OFF`
- Runtime driver: unchanged packaged Mesa 26.2.4 RADV and libdrm 2.4.133
- Build targets: `llama-server`, `llama-cli`, and `llama-bench`
- Result: PASS

The build emitted only warnings already present on the Strix Halo branch. The patched server source and all Vulkan shaders compiled and linked successfully.

## Fixed test shape

- Target: `Qwen3.8-27B-UD-Q4_K_XL.gguf`
- Drafter: `Qwen3.8-27B-DFlash2-Q4_K_M.gguf`
- Vision projector: `mmproj-F16.gguf`
- Context: 245,760 tokens, one slot, unified KV
- KV: Q8_0 K and V
- Batch and ubatch: 2,048
- Flash attention: on
- Speculation: DFlash2, maximum width 4
- RAM prompt cache: disabled for the test
- Idle-slot cache: disabled for the test
- Prompt: deterministic synthetic common prefix with divergent A/B tails
- Full prompt: approximately 9,110 tokens and spans multiple ubatches
- Output: four deterministic tokens
- Endpoint: disposable `127.0.0.1:8091`

Disabling the RAM and idle-slot caches is essential. Otherwise the 24 GB host prompt cache can hide a failed disk restore.

## Stock red baseline

Installed runtime: build 11001, commit `629d2f7b4`.

| Path | cache_n | prompt_n | Prompt time | Wall time |
|---|---:|---:|---:|---:|
| Cold A | 0 | 9,110 | 20.432 s | 20.607 s |
| Live divergent B | 7,062 | 2,050 | 4.854 s | 4.987 s |
| Restored divergent B | 0 | 9,112 | 20.785 s | 20.917 s |

The stock save returned `n_saved=9113`, `n_written=474428156`, and 82.7 ms. Restore returned HTTP success with `n_restored=9113`, `n_read=474428156`, and 42.2 ms. Despite that apparent success, the next request had `cache_n=0` and reprocessed the entire prompt. This reproduces the hybrid/recurrent silent failure.

## Candidate in-process green result

| Path | cache_n | prompt_n | Prompt time | Wall time |
|---|---:|---:|---:|---:|
| Cold A | 0 | 9,110 | 20.860 s | 21.031 s |
| Live divergent B | 7,062 | 2,050 | 4.929 s | 5.059 s |
| Restored divergent B | 7,062 | 2,050 | 4.929 s | 5.059 s |

The restored path exactly matches the live in-memory reuse boundary. Save returned `n_saved=9113`, `n_written=872152496`, and 242.2 ms. Restore returned `n_restored=9113`, `n_read=872152496`, and 126.2 ms.

The checkpoint appendix adds 397,724,340 bytes, about 379.3 MiB, for the recurrent target/draft/speculative checkpoint state.

Restore plus the required 2,050-token tail prefill took 5.186 s, approximately 4.1x faster than the 21.031 s cold path. The bounded test does not reach the brief's arbitrary 5x promotion threshold because a 2,048-token ubatch must still be replayed. For long OpenClaw prefixes, that bounded tail becomes proportionally much smaller; this must be measured rather than assumed before production promotion.

## Candidate cross-restart green result

A fresh snapshot was saved, the candidate server process was fully terminated, a new candidate process was started, and the snapshot was restored before sending divergent B.

| Measurement | Result |
|---|---:|
| Snapshot bytes | 872,152,496 |
| Restore time | 122.4 ms |
| cache_n | 7,062 |
| prompt_n | 2,050 |
| Prompt time | 5.100 s |
| Request wall time | 5.270 s |

This proves the reuse did not come from process-local cache state.

## DFlash2 and output

- Live divergent output: ` The quick brown fox`
- Restored divergent output: ` The quick brown fox`
- Live path: 2 drafted, 2 accepted
- In-process restored path: 2 drafted, 2 accepted
- In-process restored decode: 25.88 tokens/s
- Cross-restart restored path: 2 drafted, 2 accepted
- Cross-restart first decode: 19.49 tokens/s
- No drafter resynchronisation error or unexpected full prefill appeared on the fixed path.

The in-process restored output, reuse boundary, draft acceptance, and decode throughput match the live path. The lower four-token cross-restart decode number is a cold first-request measurement and is too small to classify as a throughput regression.

## Tests and limitations

- PASS: exact Strix Halo Vulkan candidate build.
- PASS: deterministic stock red reproduction on the real Qwen3.8 Q4 + DFlash2 + vision stack.
- PASS: deterministic candidate in-process restore on the same stack.
- PASS: deterministic candidate restore after a full process restart.
- PASS: upstream slot-save test file imports and collects all 17 tests.
- NOT RUN: the full upstream Python slot-save suite. Its session fixture downloads and boots every server preset, while the production-comparable candidate intentionally uses `LLAMA_CURL=OFF`. The real-model red/green tests cover the primary failure more directly, but damaged-appendix and draft-KV-mismatch unit cases remain a packaging gate.
- NOT DONE: candidate packaging or production deployment.
- NOT DONE: automatic manifest compatibility refusal for model/runtime/KV mismatches.

## Final machine state

- `qwen38-dflash2.service`: active and healthy.
- Public/Tailscale coalescing proxy health: HTTP 200.
- Production runtime: unchanged build 11001 at commit `629d2f7b4`.
- Candidate build remains at `/home/less/src/llama.cpp-tiercel/build-vk`.
- Source branch is clean and pushed only to `LeoStehlik/llama.cpp`.
- No pull request was created.
- Disposable slot snapshots were removed after their sizes, permissions, and cross-restart behavior were recorded.

## Next smallest step

Package the tested candidate in an isolated payload, run the damaged-appendix and draft-KV-mismatch unit cases plus the Strix Halo repeatability gate, then do a longer prompt benchmark that measures the practical checkpoint-size/latency tradeoff before changing production.
