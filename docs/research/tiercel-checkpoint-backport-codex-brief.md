# Brief: backport llama.cpp slot-checkpoint persistence to the Tiercel runtime branch

## FIRST: status

Mark the task working as: `Backport merged llama.cpp slot-checkpoint persistence into the Strix Halo runtime branch, prove it on Qwen3.8 + DFlash2, and open the source PR if all gates pass.`

## Confidence gate

Before starting, ask clarifying questions until you are 95% confident you have identified the correct source repository, target branch, build route and PR destination. Preserve a known-good runtime, configuration copy and rollback command before changing the Tiercel lab service.

## Task

Backport upstream llama.cpp commit `033df86b69ec1a333eb241f0c16325a4d43dcff5` from merged PR [#26004](https://github.com/ggml-org/llama.cpp/pull/26004) into the source branch used by Tiercel's Strix Halo runtime, build it in isolation, prove real slot reuse with Qwen3.8-27B Q4 and DFlash2, then open a PR against that source branch.

## Known environment

- Tiercel deployment repo: `/home/less/qwen38-dflash2`
- Deployment repo remote: `LeoStehlik/qwen3.8-27b-strix-halo-dflash2`
- Packaged runtime: `v0.8.0`, built from llama.cpp commit `629d2f7b45285e3f7556648c50c167159c532476`
- Runtime source fork documented by the deployment: `Nathanw1014/llama.cpp`
- Runtime source branch documented by the deployment: `strix-halo-vulkan`
- Upstream fix: llama.cpp PR #26004, merged as `033df86b69ec1a333eb241f0c16325a4d43dcff5`
- Upstream fix touches:
  - `common/common.cpp`
  - `common/common.h`
  - `examples/speculative-simple/speculative-simple.cpp`
  - `tools/server/server-context.cpp`
  - `tools/server/tests/unit/test_slot_save.py`
  - `tools/server/tests/utils.py`

The deployment repository packages binaries and configuration. The code PR belongs in the llama.cpp source fork, not the deployment repo. Verify this before proceeding. If the actual source or intended PR target differs, stop and ask Leo.

## Read first

- `/home/less/qwen38-dflash2/runtime/MANIFEST.txt`
- `/home/less/qwen38-dflash2/runtime/README.md`
- `/home/less/qwen38-dflash2/docs/REFERENCES.md`
- `/home/less/tiercel-slot-save-restore-codex-brief.md`
- `/home/less/tiercel-slot-save-restore-codex-review.md`
- <https://github.com/ggml-org/llama.cpp/pull/26004>
- <https://github.com/ggml-org/llama.cpp/issues/25913>

Read the source repository's root and nearest scoped `AGENTS.md` and contribution instructions before editing.

## Execution

### Phase 1: discover and freeze the base

1. Locate the source clone that produced the packaged runtime, or clone `Nathanw1014/llama.cpp` without touching the deployment repo.
2. Fetch both the fork and `ggml-org/llama.cpp`.
3. Verify the intended target is `strix-halo-vulkan` and record its current SHA.
4. Prove whether the target already contains `033df86b` with `git merge-base --is-ancestor`.
5. Inspect PR #26004 state through the GitHub API and confirm it is merged.
6. Create a fresh branch and isolated worktree from the current target branch. Suggested branch: `fix/slot-checkpoint-persistence`.
7. Keep all builds and long tests in tmux. Preserve the packaged runtime as the known-good rollback target.

If the target already contains the fix, do not cherry-pick it. Reproduce the failure and identify why the packaged build differs.

### Phase 2: reproduce the stock failure

Use the existing packaged Tiercel runtime as the red baseline. Tiercel is an experimental workbench, so the service may be stopped, restarted or temporarily repointed after its current runtime, configuration and rollback command are captured.

Run the deterministic approximately 8K-token Qwen3.8 Q4 test from the corrected slot-save brief:

1. Start with the existing Q8 KV, one slot, DFlash2 and template-relevant flags, adding a temporary `--slot-save-path`.
2. Pin requests to `id_slot: 0`, set `cache_prompt: true`, use deterministic sampling and generate only a short output.
3. Cold-prefill, save, erase, restore, then send the identical prefix plus a short suffix.
4. Record `timings.cache_n`, `timings.prompt_n`, prompt time, TTFT, output, DFlash drafted/accepted counts and relevant logs.

Expected red result on the packaged runtime: restore reports success but `cache_n` is near zero and the full prefix is reprocessed.

### Phase 3: backport

1. In the isolated source worktree, attempt:

   `git cherry-pick -x 033df86b69ec1a333eb241f0c16325a4d43dcff5`

2. Resolve conflicts semantically against the current Strix Halo branch. Preserve all six upstream file changes unless current code proves one obsolete.
3. Do not rewrite the solution as a sidecar design. PR #26004 appends a tagged checkpoint payload to the existing slot file and includes draft-state and corruption handling.
4. Preserve upstream attribution and the `-x` reference.
5. Keep the patch limited to the backport and necessary conflict adaptation.

### Phase 4: build and source tests

1. Build using the source fork's documented Strix Halo Vulkan toolchain.
2. Run the slot-save test file, including restore reuse, restart reuse, damaged appendix and draft KV mismatch coverage.
3. Run the owning server test inventory and any repository-mandated formatting or build checks.
4. Record exact commands and results against the final head.

Stop on unexplained test failure. Do not weaken or remove upstream tests to make the backport pass.

### Phase 5: Tiercel behavioral proof

Run the candidate binary from the isolated build directory. It may replace the active lab process temporarily, provided the known-good runtime remains intact and rollback is tested afterward.

Repeat the exact Phase 2 fixture:

1. In-process save, erase and restore.
2. Full candidate child restart, restore and repeat.
3. Verify the restored request uses nearly the full saved prefix through `cache_n`, with only the tail in `prompt_n`.
4. Compare restored-path latency with cold prefill.
5. Verify deterministic output, DFlash drafted/accepted counts, acceptance ratio, decode throughput and absence of drafter resynchronisation errors.
6. Confirm the current packaged binary still runs unchanged after the test.

### Phase 6: PR

Only after all acceptance criteria pass:

1. Push the clean branch to an authorised fork.
2. Open a PR against the verified source target branch, expected to be `Nathanw1014/llama.cpp:strix-halo-vulkan`.
3. Keep the PR narrowly framed as a backport of merged upstream llama.cpp PR #26004.
4. Include base/head SHAs, upstream merge SHA, conflict adaptations, red/green Tiercel evidence, source test results and residual risks.
5. Do not publish a new packaged release in this task. Temporary lab wiring needed for proof is allowed and must be reverted or committed separately.

## Acceptance criteria

### AC1: ancestry and target are proven

The result names the source repo, target branch, base SHA and confirms whether `033df86b` was absent by Git ancestry.

### AC2: stock runtime reproduces the defect

The 8K save/restore fixture reports successful restore followed by near-zero `cache_n` or full-prefix `prompt_n`.

### AC3: backport preserves the upstream fix

The final diff contains the checkpoint appendix, safe load/write handling, draft mismatch behavior and associated tests from #26004, with only documented conflict adaptations.

### AC4: source tests pass

The final head passes the slot-save suite, owning server tests and required build checks without weakening tests.

### AC5: Qwen3.8 reuse works in process and after restart

On the candidate Tiercel binary, `cache_n` covers nearly the full saved prefix and `prompt_n` contains only the short appended tail in both cases.

### AC6: DFlash2 remains healthy

Candidate output remains correct and deterministic. Draft acceptance and decode throughput show no unexplained regression or resynchronisation failure.

### AC7: the lab remains recoverable

The known-good runtime and model files remain intact. Any temporary service or configuration change is documented, and the original runtime can be restored with the tested rollback command.

### AC8: PR is reviewable

The PR targets the verified source branch, links upstream #26004, contains exact-head proof and has no unrelated changes.

## Constraints

- Work in an isolated source worktree and disposable server process.
- Tiercel is a lab machine. Restarting or temporarily repointing its service is allowed after capturing the known-good state and rollback.
- Do not edit OpenClaw configuration.
- Do not change models, quantisation, context, KV types, DFlash2 settings or benchmark fixture between red and green tests.
- Bind candidate endpoints to localhost only.
- Do not commit model files, slot snapshots, generated binaries or private prompt material.
- Preserve at least 20 GiB free disk space.
- Treat slot snapshots as sensitive and remove disposable snapshots after testing.
- Stop before PR creation if the source target is ambiguous, the branch is dirty, the fix cannot be isolated, or any AC fails.

## Non-goals

- Packaging or deploying the fixed runtime.
- Changing the router, OpenClaw or Agent Body design.
- Running the full 150K OpenClaw prompt before the 8K proof passes.
- Reimplementing a different checkpoint format.
- Opening a second PR against upstream `ggml-org/llama.cpp`.

## Deliver

- Source repo, target branch, base SHA and head SHA.
- Changed files and conflict adaptations.
- Stock red and candidate green evidence.
- Source test/build results.
- Tiercel in-process and restart results.
- DFlash2 correctness and performance result.
- PR URL, title and body if all gates pass.
- Residual risks and the next separate step for packaging/deployment.

Suggested PR title:

`server: backport persistent slot checkpoints from upstream #26004`

## LAST: status

Mark the task done only after the PR is opened or a concrete stop condition is reached. Report the PR URL or blocker, exact tested head, final lab state, tested rollback and next smallest step.
