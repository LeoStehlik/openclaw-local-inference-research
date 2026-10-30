# OpenClaw local inference research

OpenClaw can build a very capable agent. It can also hand a local model a 150K-token cold prompt before the agent says a word.

This repository is research into keeping the capable agent while removing repeated prompt computation. Emptying `SOUL.md`, hiding tools and throwing memory away would make the benchmark faster. It would also miss the point.

## The observed problem

The first live test used:

- OpenClaw 2026.10.6;
- Qwen3.8-27B Q4 on llama.cpp with DFlash2;
- a Ryzen AI Max+ 395 machine with 128GB unified memory;
- a 245,760-token context;
- Q8 KV, prompt caching and one inference slot.

OpenClaw produced a 152,781-token cold request.

- First response: 9m32s
- Ten cached follow-ups: 47.5s median, 53.6s mean
- Eight of ten cached turns: under one minute
- All ten cached turns: under two minutes
- A second unrelated large session displaced the hot sequence and blocked work behind it

The model server stayed healthy. The same agent became usable once its prefix was warm. The expensive part was repeatedly creating and losing model state for a body that was mostly unchanged.

See [the sanitized live finding](docs/findings/2026-10-30-tiercel-live-trace.md).

## The useful distinction

Three artifacts are easy to conflate:

1. Canonical prompt text
2. Token IDs
3. Computed KV state

Saving text or token IDs avoids parsing and tokenisation. Those are cheap. Saving KV state avoids running every token through every transformer layer again.

The working question is:

> Can OpenClaw compile the durable part of an agent into a versioned KV artifact, then begin each local session by restoring that body and computing only the changing tail?

## Current hypotheses

### A. Long coding sessions are warm sequences

Codex and OpenCode may have access to the same agent and memory files without injecting all available content into every cold request. A long coding session grows incrementally from a smaller prefix and benefits from normal prefix reuse.

This needs a controlled comparison using the same model endpoint, workspace and task. See [OpenCode versus OpenClaw](docs/research/opencode-openclaw-comparison.md).

### B. Persistent slot state removes repeated body prefill

llama.cpp can save and restore slot state. If a 100K-plus body restores far faster than it prefills, this is the strongest near-term primitive we have found.

The first experiment is deliberately narrow. It measures snapshot size, save time, restore time, appended-tail latency and correctness before any OpenClaw integration.

See [the Tiercel slot experiment brief](docs/research/tiercel-slot-save-restore-codex-brief.md).

### C. Cloud hardware could compile the body

A self-controlled cloud GPU could run the exact GGUF and a compatible llama.cpp revision, prefill a stable Agent Body, export the slot state and send it to the local machine. This is plausible as an occasional body-build pipeline. Live WAN transfer per request is probably defeated by multi-gigabyte KV state.

CUDA-to-Vulkan slot portability remains unproven. See [cloud prefill, local decode](docs/research/cloud-prefill-local-decode-hypothesis.md).

### D. OpenClaw needs deterministic body assembly

Cache reuse requires identical leading tokens. Stable rules, identity and tools need deterministic ordering before timestamps, request IDs, retrieved context and current conversation.

The proposed compiler creates ordered segments, hashes their exact token form and rebuilds only from the changed segment onward. See [Agent Body compiler](docs/architecture/agent-body-compiler.md).

## Research order

1. Prove same-process slot save and restore.
2. Prove restore after service restart.
3. Compare cold OpenCode and OpenClaw request composition.
4. Prove deterministic stable-prefix assembly with a disposable fixture.
5. Restore that fixture before one local OpenClaw session.
6. Test cross-process and cross-backend slot portability.
7. Only then design session affinity, eviction protection or a KV lifecycle manager.

The full sequence, promotion gates and stop conditions live in the [research plan](docs/research/research-plan.md).

## What this project will not do

- Empty agent identity or hard rules to win a latency benchmark
- Hide tool capability from the model without measuring the loss
- Claim semantically similar text can share exact KV state
- Change frontier-provider behaviour by default
- Build a router before the cache primitive is proven
- Publish private prompts, memory or infrastructure details

## Repository map

- `docs/findings/`: sanitized observed results
- `docs/architecture/`: candidate architecture, still provisional
- `docs/research/`: hypotheses, briefs and experiment plans
- `docs/decisions/`: invariants that constrain the work

## Status

Research only. No OpenClaw implementation has been selected.

The current best candidate for a large win is persistent KV restoration of a deterministic base body. It still has to survive measurement on the real machine.
