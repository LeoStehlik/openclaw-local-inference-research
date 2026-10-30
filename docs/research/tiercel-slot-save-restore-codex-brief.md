# Brief: Tiercel Qwen3.8 slot save/restore falsification

## FIRST: status

Mark the task working and state that this is a bounded 8K-token KV-persistence falsification test. Do not combine it with OpenClaw prompt redesign, router architecture or production deployment.

## Confidence gate

Before starting any work, ask clarifying questions until you are 95% confident you can complete this task safely. If already confident, state assumptions and proceed.

## Task

Prove or falsify whether Tiercel's installed llama.cpp build can save and restore usable Qwen3.8-27B Q4 KV state, both within one child process and after a child restart.

## Context

Tiercel runs `qwen38-dflash2.service` with Qwen3.8-27B Q4/Q5 presets. The active Q4 child uses prompt caching, idle-slot caching, Q8 KV, a 245,760-token context, one slot and DFlash2. It does not configure `--slot-save-path`.

Qwen3.8 identifies as the hybrid/recurrent `qwen35` architecture. Upstream llama.cpp has a known silent failure for this model class: save and restore return success, but context checkpoints are absent and the next request fully re-prefills. HTTP 200, matching `n_restored`, a populated slot and fast file reading are therefore insufficient evidence.

The decisive evidence is the completion after restoration: `timings.cache_n` must cover nearly the entire saved prefix and `timings.prompt_n` must contain only the appended tail.

## Read first

- This brief and `tiercel-slot-save-restore-codex-review.md` in `/home/less`.
- The exact installed llama.cpp server help and server README for Tiercel's pinned build.
- The current service, router configuration, active child command and model preset files.
- <https://github.com/ggml-org/llama.cpp/issues/28194>
- <https://github.com/ggml-org/llama.cpp/issues/26676>

Do not assume flags or behavior from another llama.cpp build.

## Environment discovery

1. Use the existing authorised Tiercel operator route. Keep private connection details out of reports.
2. Locate the source-controlled deployment repository and exact configuration owning the active child flags.
3. Record git status and preserve unrelated changes. Stop if overlapping edits cannot be preserved.
4. Confirm the installed build exposes `--slot-save-path` and the save, restore and erase slot endpoints.
5. Discover the dynamically allocated child-server loopback port from verified router status or the active command. Do not send slot operations to the public coalescer or router.
6. Record runtime commit, model hash, active flags, context size, KV types, flash-attention state, slot count and available disk space.
7. Capture baseline health and one small ordinary completion.

## Phase A: cheap go/no-go test

Use a bounded maintenance window or a disposable localhost child that does not compete with production for unified memory. Run the exact Q4 target with its current Q8 KV, DFlash2, context and template-relevant settings, adding only a dedicated `--slot-save-path`.

Use a deterministic synthetic prompt of approximately 8K tokens. It must exceed multiple 2,048-token ubatches. Do not use Francis's raw prompt.

Pin every request to `id_slot: 0`, set `cache_prompt: true`, use deterministic sampling and request only a short output.

Sequence:

1. Cold-prefill the synthetic prefix and record `prompt_n`, `cache_n`, prompt time, TTFT and output.
2. Confirm slot 0 is idle and contains the expected prompt-token count.
3. Save slot 0 and record `n_saved`, `n_written`, save time, apparent size and disk usage.
4. Erase slot 0 and confirm it is empty.
5. Restore slot 0 and record `n_restored`, `n_read` and restore time.
6. Send the identical prefix plus a short deterministic suffix.
7. Inspect the resulting `timings.cache_n` and `timings.prompt_n`.

### Immediate stop condition

If restore returns success but `cache_n` is near zero or the full prefix appears in `prompt_n`, classify explicit slot persistence as non-functional for Qwen3.8 on this build. Stop. Do not run a 150K test and do not add `slot-save-path` to the production preset.

## Phase B: cross-process proof

Run only if Phase A demonstrates real KV reuse.

1. Save a fresh snapshot.
2. Fully restart the model child so its original in-memory state is gone.
3. Rediscover the child's loopback port.
4. Restore the snapshot and send the identical prefix plus short suffix.
5. Verify `cache_n`, `prompt_n`, output correctness and restored-path latency again.

## Phase C: production configuration

Run only if both Phase A and Phase B pass and the restored path meets the promotion threshold.

1. Add the dedicated slot-save path through the source-controlled preset or service configuration that owns the child flags.
2. Preserve every model, DFlash2, vision, prompt-cache, KV and router flag.
3. Restart only the affected user service.
4. Verify Q4 autoload, Q4/Q5 listing, health and an ordinary completion.
5. Repeat the synthetic save/erase/restore check through the active child endpoint.

Do not automate restore on startup in this task.

## Acceptance criteria

### AC1: owning configuration and runtime identity are known

The report identifies the source-controlled setting owning `--slot-save-path` and records the runtime identity needed to prevent incompatible restores.

### AC2: real in-process KV reuse is proven

After save, erase and restore, `cache_n` covers nearly the full saved prefix and `prompt_n` contains only the short suffix. API success alone is a failure of proof.

### AC3: cross-process KV reuse is proven

AC2 passes after a full child restart and port rediscovery.

### AC4: restored path is materially cheaper

Compare:

- `restore time + tail prefill + TTFT`
- `cold full-prefix prefill + TTFT`

The restored path must be at least five times faster. Report save cost separately.

### AC5: DFlash2 remains healthy

After successful restoration, record output correctness, drafted and accepted counts, acceptance ratio, decode throughput and any drafter resynchronisation or full-prefill logs. Reject promotion on unexplained drift or destroyed speculative-decoding performance.

### AC6: incompatible snapshots are refused without loading

Create a manifest containing model hash, runtime commit, context size, KV types, flash-attention state, slot count and checkpoint settings. Use model-specific names or directories. Demonstrate refusal when a copied manifest field differs. Do not cross-load Q4 into Q5 and do not corrupt a snapshot.

### AC7: production behavior and rollback survive

If Phase C is reached, existing Tiercel behavior remains available and removing the one source-controlled path change restores the previous command and behavior.

## Storage safeguards

- Use one narrowly scoped owner-only directory outside Git.
- Directory mode `0700`, process `umask 077`, snapshot files `0600`.
- Keep one snapshot initially and at least 20 GiB free.
- Treat snapshots as sensitive prompt-derived state.
- Never commit snapshots to Git.

## Constraints

- Preserve unrelated user changes and deployment ownership.
- No OpenClaw changes, router redesign, session manager or automatic prewarming.
- No model, quantisation, context, KV type, DFlash2 or sampler changes.
- Bind experimental endpoints to localhost only.
- No private agent prompts in fixtures or reports.
- Stop on OOM, repeated crash, corruption, unexplained output drift or insufficient disk space.
- Do not apply an upstream patch or alternate llama.cpp fork in this task.

## Non-goals

- A 150K benchmark unless the 8K test passes.
- Multi-session scheduling or Agent Body compilation.
- Proving compatibility across llama.cpp upgrades.
- Achieving the final 60-to-120-second OpenClaw target.
- Applying or publishing an upstream fix.

## Deliver

Write `docs/research/tiercel-slot-save-restore.md` in the deployment repository with:

- PASS/FAIL/UNKNOWN for each AC;
- measured timings and `cache_n`/`prompt_n` evidence;
- snapshot size and storage behavior;
- DFlash2 behavior if restoration works;
- configuration change and rollback, if Phase C is reached;
- one verdict: reject stock persistence, investigate the upstream checkpoint fix in a disposable build, or promote to the next Agent Body experiment.

## LAST: status

Mark the task done only when every reached phase has a clear verdict. Notify the orchestrator with the result path, changed files, tests run, rollback status and the next smallest justified experiment.
