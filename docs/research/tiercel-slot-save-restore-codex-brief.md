# Brief: Tiercel llama.cpp slot save/restore experiment

## FIRST: status

Mark the task working and state that this is a bounded KV snapshot experiment. Do not combine it with OpenClaw prompt redesign, router architecture or model tuning.

## Confidence gate

Before starting any work, ask clarifying questions until you are 95% confident you can complete this task safely. If already confident, state assumptions and proceed.

## Task

Discover, enable and benchmark llama.cpp's explicit slot save/restore support on Tiercel, proving whether a large prepared KV state can replace repeated cold prefill for the same model and prompt prefix.

## Context

Tiercel currently runs `qwen38-dflash2.service` with Qwen3.8-27B Q4/Q5 model presets. The active Q4 child server has prompt caching, idle-slot caching, a 24GB RAM prompt cache, Q8 KV, a 245,760-token context and one slot. It does not currently configure `--slot-save-path`.

The live OpenClaw trace showed a 152,781-token cold request taking 9m32s end to end, followed by much faster cached turns. A second unrelated large session displaced the hot sequence. This experiment asks whether explicit saved slot state is the near-term reusable primitive.

## Read first

- this repository's `README.md`;
- `docs/architecture/agent-body-compiler.md`;
- `docs/research/research-plan.md`;
- the exact installed llama.cpp server help and server README for the Tiercel build;
- the current `qwen38-dflash2.service`, router configuration and model preset files.

Do not rely on flags from a different llama.cpp build.

## Environment discovery

1. Use the existing authorised operator route to Tiercel. Do not record private connection details in this repository.
2. Locate the source-controlled deployment repository, service unit, router configuration, active child command and model preset responsible for `qwen38-dflash2.service`.
3. Record the current git status and stop if overlapping uncommitted changes cannot be preserved.
4. Confirm the active build exposes:
   - `--slot-save-path`;
   - `GET /slots`;
   - `POST /slots/{id}?action=save`;
   - `POST /slots/{id}?action=restore`;
   - `POST /slots/{id}?action=erase`.
5. Confirm available storage and choose a dedicated, owner-writable cache directory outside the repository. Do not use a broad home-directory target.
6. Capture baseline service health and one small ordinary completion before changing configuration.

## Execution

### Phase A: disposable proof before persistent configuration

If the active child cannot expose slot persistence without restart, launch a disposable server on a different localhost port using the same Q4 model, template-relevant flags and a reduced but representative context. Do not compete with the production endpoint for memory if both cannot coexist safely.

Use a synthetic deterministic prompt large enough to make prefill timing measurable. Do not use Francis's private raw prompt as the benchmark fixture.

Measure:

- cold prompt tokens and prefill time;
- generated tokens and decode time;
- snapshot size and save time;
- restore time;
- time to first token after restoring and appending a new suffix;
- output correctness against the cold control;
- behaviour after erase and intentional cache miss.

### Phase B: enable the storage path

Only after Phase A proves the endpoint and file format work:

1. add a dedicated slot-save path through the source-controlled preset or service configuration that actually owns the child server flags;
2. preserve every existing model, DFlash2, vision, prompt-cache, KV and router flag;
3. restart only the affected user service;
4. verify Q4 loads, health returns, the model list remains correct and a normal completion succeeds;
5. repeat save, erase and restore through the production localhost endpoint using a disposable synthetic slot state.

Do not automate restore on startup in this task.

## Acceptance criteria

### AC1: exact owning configuration is identified

The report names the repository file and setting that controls the active child server's slot-save path. No installed generated file is treated as the source of truth.

Verify: service restart reproduces the expected active command with the dedicated slot-save path present.

### AC2: save and restore work

A deterministic synthetic prompt is prefetched into the single slot, saved, erased and restored without an API error or server crash.

Verify: the restored slot accepts a new suffix and produces the expected grounded response.

### AC3: restore is materially cheaper than cold prefill

Report cold prefill time, snapshot save time, snapshot size, restore time and restored time to first token separately.

Promotion threshold: restore plus appended-tail prefill is at least 5 times faster than the equivalent cold prefill. If it is slower, report the result and stop without designing around it.

### AC4: cache identity boundaries are tested

At minimum, prove and document expected failure or invalidation for one incompatible identity, such as another quant, changed model fingerprint or changed prompt prefix. Never restore a snapshot into a knowingly incompatible model process.

### AC5: existing Tiercel service behaviour survives

Q4 autoload, Q4/Q5 model listing, DFlash2, vision configuration, prompt caching, metrics and ordinary generation remain available after the scoped configuration change.

### AC6: rollback is proven

Removing the one slot-save-path configuration change and restarting the affected service restores the prior command and behaviour. Benchmark snapshots can be removed independently after the experiment.

## Constraints

- Preserve existing user changes and deployment ownership.
- No OpenClaw configuration or code changes.
- No router redesign, session manager or automatic prewarming.
- No model, quantisation, context, KV type, DFlash2 or sampler changes.
- No public network exposure.
- Bind experimental endpoints to localhost only.
- Do not copy private agent prompts into the research repository or benchmark report.
- Do not claim semantic equivalence permits KV reuse.
- Stop on OOM, repeated server crash, model corruption, unexplained output drift or insufficient disk space.

## Non-goals

- Solving multi-session scheduling.
- Building the Agent Body compiler.
- Proving snapshots survive arbitrary llama.cpp upgrades.
- Achieving the final 60-to-120-second OpenClaw target.
- Publishing or opening an upstream PR.

## Deliver

- `docs/research/results/tiercel-slot-save-restore.md` containing configuration category, commands with private connection details removed, timings, snapshot size, correctness results and verdict;
- the minimal source-controlled slot-save-path change if AC1 to AC5 pass;
- exact rollback steps;
- a clear recommendation: promote to Agent Body fixture research, revise the experiment, or reject persistent snapshots.

## LAST: status

Mark the task done only after the scoped acceptance criteria have verdicts. Notify the orchestrator with the result file, changed files, tests run, rollback status and the next smallest justified experiment.
