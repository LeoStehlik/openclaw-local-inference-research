# OpenClaw Local Inference Research

Research into preserving OpenClaw's full agent body while making established agents usable on local inference hardware.

## Problem

An established OpenClaw agent can assemble a valid 100K-plus-token request from identity, rules, tools, memory, runtime context and conversation history. Frontier providers hide much of the repeated prefill cost. A local OpenAI-compatible endpoint usually receives the same large textual body and must compute it again whenever prefix reuse is lost.

The first Tiercel live trace established the shape of the problem:

- a 152,781-token cold request took 9m32s end to end;
- stable cached follow-ups had a 47.5s median;
- an unrelated large session displaced the hot sequence;
- the model and hardware remained healthy throughout.

The research question is therefore:

> Can OpenClaw preserve the complete agent body while converting most repeated body prefill into a reusable, versioned KV artifact, leaving only a bounded per-session tail to compute?

## Principles

- Preserve identity, rules, tools, memory routes and safety boundaries.
- Optimise uncached computation, not headline context size.
- Do not mistake tokenisation for transformer prefill.
- Keep frontier-provider behaviour unchanged by default.
- Measure existing llama.cpp primitives before designing new infrastructure.
- Prefer one decisive experiment over a speculative platform rewrite.

## Current work

1. [Agent Body compiler concept](docs/architecture/agent-body-compiler.md)
2. [Tiercel slot save/restore Codex brief](docs/research/tiercel-slot-save-restore-codex-brief.md)
3. [Research plan](docs/research/research-plan.md)
4. [Decision: preserve the body, change its loading lifecycle](docs/decisions/0001-preserve-agent-body.md)

## Current hypothesis

The highest-leverage near-term experiment is a persistent llama.cpp KV snapshot for a large, deterministic Agent Body. If restoring that snapshot is materially faster than recomputing it, new OpenClaw sessions can start from the prepared body and append only session-specific context.

No implementation architecture is selected until that experiment produces timings, correctness results and operational constraints.
