# Review: Tiercel llama.cpp slot save/restore experiment

## Verdict

The brief is well structured, but it should be revised before execution.

There is one critical issue: Qwen3.8 reports the `qwen35` architecture with
SSM/recurrent metadata, and upstream llama.cpp has a known silent failure where
slot restore reports success while hybrid models fully re-prefill because
context checkpoints are not persisted.

Upstream evidence:

- [Hybrid/recurrent slot restore loses prompt reuse](https://github.com/ggml-org/llama.cpp/issues/28194)
- [Slot restore can report success while providing no KV reuse](https://github.com/ggml-org/llama.cpp/issues/26676)
- [Pinned server documentation for build commit `629d2f7b4`](https://github.com/ggml-org/llama.cpp/blob/629d2f7b4/tools/server/README.md)

The experiment remains worthwhile, but it should begin as a small, fast
falsification test rather than a large-context deployment exercise.

## Verified locally

- The installed build exposes `--slot-save-path`.
- `GET /slots` works on the active child server.
- `POST /slots/0?action=save` currently returns HTTP 501 with the expected
  explanation that the server must start with `--slot-save-path`.
- The documented save, restore and erase endpoint contract matches the pinned
  llama.cpp commit.
- The Qwen3.8 target contains `qwen35` architecture and SSM metadata, placing it
  in the model class affected by the known checkpoint-persistence problem.
- The machine currently has approximately 43 GiB free on the relevant
  filesystem.
- `docs/architecture/agent-body-compiler.md` and
  `docs/research/research-plan.md` do not exist under
  `/home/less/qwen38-dflash2`.
- Slot operations must target the dynamically allocated loopback child-server
  port. The router's `/slots` endpoint does not expose the model child's slot.

## Required changes to the brief

### 1. Make Phase A a fast falsification test

Start with a deterministic prompt of approximately 8K tokens, large enough to
span more than one 2,048-token ubatch and exercise context checkpoints. Do not
start with the 150K-token OpenClaw-sized prefix.

If the restored request fully re-prefills, stop immediately and reject explicit
slot persistence on this build. There is no value in repeating the same failure
at 150K tokens.

### 2. Measure actual cache reuse

The decisive values are returned by the completion after restoration:

```text
timings.cache_n
timings.prompt_n
```

A successful HTTP response, matching `n_restored`, restored token count in
`GET /slots`, or short file-read time does not prove that the restored KV is
usable. The known failure mode reports success and then silently reprocesses
the full prompt.

Use this sequence:

```text
cold deterministic prefill
save slot 0
erase slot 0
restore slot 0
send the identical prefix plus a short suffix
inspect cache_n and prompt_n
```

Pin the requests with `id_slot: 0`, enable `cache_prompt: true`, use
deterministic sampling, and request only a short output.

The restored request passes only if `cache_n` covers nearly the entire saved
prefix and `prompt_n` contains only the short appended tail.

### 3. Test both in-process and cross-process restoration

If save, erase and restore work within one child process, save again and repeat
the restoration after a full child-server restart.

Cross-process restoration is essential because Q4 to Q5 to Q4 router switching
destroys the original Q4 child process. An in-process-only result does not prove
the intended persistence use case.

### 4. Use the child server endpoint

Replace references to the “production localhost endpoint” with the active
model child's ephemeral loopback endpoint.

The production request path is:

```text
public coalescer -> router -> dynamically allocated model child
```

Slot state belongs to the child. Discover its current loopback port from the
router model-status data or the verified active command before each operation.
Do not expose the child endpoint beyond localhost.

### 5. Resolve the missing research paths

The two required research files named in the brief are not present in the
deployment repository. Either specify the separate research repository and its
absolute local path, or remove those prerequisites.

If the result belongs in `qwen38-dflash2`, a suitable destination would be:

```text
docs/research/tiercel-slot-save-restore.md
```

Create that directory only when executing the experiment and producing the
result.

### 6. Correct the incompatible-identity acceptance criterion

AC4 currently asks for an incompatible restore test while also saying never to
restore a knowingly incompatible snapshot. Replace it with a non-destructive
identity-boundary test:

- Store snapshots in model-specific directories or use model-specific names.
- Record the target model SHA-256, runtime commit, context size, KV types,
  flash-attention state, slot count and relevant checkpoint settings.
- Refuse restoration when any identity field differs.
- Do not deliberately feed a Q4 snapshot to Q5 or corrupt a snapshot merely to
  observe the failure mode.

### 7. Add storage and permission safeguards

Use a dedicated owner-only directory outside Git, for example a narrowly scoped
subdirectory under the user's cache directory.

Required safeguards:

- Directory mode `0700`
- Process `umask 077`
- Snapshot files mode `0600`
- One snapshot initially
- Preserve at least 20 GiB of free filesystem space
- Record actual `n_written`, apparent size and disk usage
- Never add snapshots to Git

KV state can encode information derived from private prompts even when the
first benchmark fixture is synthetic, so snapshot storage should be treated as
sensitive by default.

### 8. Verify DFlash2 after restoration

Slot persistence primarily concerns the target state. Do not assume the draft
model's required state is restored equivalently.

After a successful cache-reuse check, record:

- Generated output correctness
- DFlash drafted and accepted counts
- Draft acceptance ratio
- Decode throughput
- Any drafter resynchronisation or full-prefill log messages

The restored route should not be promoted if it technically reuses the target
prefix but destroys speculative-decoding performance or produces unexplained
output drift.

### 9. Make the promotion threshold explicit

The existing five-times-faster threshold is sensible, but calculate it from
the complete restored path:

```text
restore time + appended-tail prefill + time to first generated token
```

Compare that with:

```text
cold full-prefix prefill + time to first generated token
```

Report snapshot save time separately because saving affects eviction cost even
when it does not affect the later restore latency.

### 10. Stop on the known silent failure

Add this explicit stop condition:

> If restore returns success but the next identical-prefix request reports
> `cache_n` near zero or reprocesses the full prompt, classify the endpoint as
> non-functional for Qwen3.8 on this build. Do not add `slot-save-path` to the
> persistent production preset.

## Recommended Phase A

Use a bounded maintenance window or a disposable localhost child that does not
compete with production for unified memory.

1. Capture the runtime commit, target hash, active flags, available disk space
   and baseline health.
2. Start the exact Q4 target with the existing Q8 KV, DFlash2, context and
   template-relevant settings plus a dedicated `--slot-save-path`.
3. Generate a deterministic synthetic prompt of approximately 8K tokens.
4. Cold-prefill it and record `prompt_n`, `cache_n`, prompt time and output.
5. Confirm slot 0 is idle and contains the expected prompt-token count.
6. Save slot 0 and record `n_saved`, `n_written`, save time and file size.
7. Erase slot 0 and confirm it is empty.
8. Restore the snapshot and record `n_restored`, `n_read` and restore time.
9. Send the identical prefix plus a small deterministic suffix, pinned to slot
   0 with prompt caching enabled.
10. Inspect `cache_n` and `prompt_n` before considering the test successful.
11. If reuse works, repeat after a full child restart.
12. Only if both cases work and exceed the five-times threshold should Phase B
    modify the source-controlled production preset.

## Expected result

Given the model architecture and known upstream behavior, the likely result is:

```text
save:       HTTP success
restore:    HTTP success with n_restored matching n_saved
next call:  cache_n approximately zero; full prompt reprocessed
verdict:    reject explicit slot persistence on this build
```

That negative result would still be valuable. It would distinguish a broken
on-disk restore path from the already-proven healthy in-memory prefix cache and
avoid investing in session orchestration around an endpoint that only appears
to work.

## Recommendation

Proceed only as the approximately 8K-token go/no-go experiment described above.
Do not make a persistent production change unless actual post-restore cache
reuse is demonstrated both in-process and after child restart.
