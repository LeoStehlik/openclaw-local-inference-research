# Research plan

## Goal

Find the smallest architecture that removes at least 90 percent of repeated stable-body prefill without weakening the OpenClaw agent.

## Questions in order

1. Can Tiercel save and restore a 100K-plus llama.cpp slot state correctly?
2. What are snapshot size, save time, restore time and time to first token after restore?
3. Does a restored base state safely accept a different session tail?
4. Does the state survive idle-slot eviction and service restart?
5. Which model, quant, tokenizer, template and server-build changes invalidate it?
6. Can OpenClaw produce an identical stable prefix across two new sessions?
7. How much of the 152K live request is genuinely stable body, session history, memory hydration, tools and runtime material?
8. What belongs in prepared KV, proactive retrieval or the live tail?
9. Is a llama.cpp slot snapshot portable across matched CUDA and Vulkan deployments, allowing cloud prefill and local decode?

## Iterations

### R1: Existing primitive

Enable and benchmark llama.cpp slot save/restore on Tiercel. No OpenClaw code changes.

Promotion gate: restored time to first token is materially lower than cold prefill and output remains correct.

### R2: Deterministic fixture

Construct a disposable large agent-body fixture and prove that two separately assembled sessions produce an identical token prefix.

Promotion gate: byte and token hashes match; volatile runtime material begins after the reusable boundary.

### R3: Minimal integration

Add a narrow experimental path that restores the prepared body before dispatching one disposable local session.

Promotion gate: new session starts inside 120 seconds with capability and identity checks passing.

### R4: Segmentation and retrieval

Measure which context must stay resident and which can be paged without capability loss.

Promotion gate: reduced uncached tail with no material regression against the full-context control.

### R5: Lifecycle and scheduling

Only after R1 to R4, design protection, session affinity, eviction and concurrency policy.

### Parallel hypothesis after R1: cloud-built base state

If local save/restore passes, test state portability in increasing order of distance: same process, restart, second local process, different backend, then a temporary self-controlled cloud CUDA worker. Do not attempt live WAN KV transfer before offline snapshot portability is proven.

## Stop conditions

Stop and reconsider the architecture if:

- snapshot restore approaches cold-prefill time;
- snapshots are too large for practical RAM or NVMe residency;
- restored state is not reliably bound to model and template identity;
- session forking corrupts output or leaks state between sessions;
- deterministic OpenClaw assembly cannot be achieved without changing frontier behaviour;
- capability tests show that context paging materially dumbs down the agent.

## Evidence discipline

Keep raw traces and private prompts outside the public research repository. Publish aggregate timings, configuration categories, reproducible synthetic fixtures and redacted failure descriptions.
