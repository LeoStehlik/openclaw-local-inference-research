# Hypothesis: cloud-built KV state for local decode

## Question

Can a self-controlled cloud GPU prefill a large deterministic Agent Body, export the resulting llama.cpp slot state, and allow Tiercel to restore that state for local generation?

## Short answer

Plausible as an occasional body-build pipeline, unproven across CUDA and Vulkan, and unlikely to be useful as a per-request service over an ordinary WAN link.

Commercial model APIs do not expose their internal KV cache. The cloud side would need to run a controlled llama.cpp deployment using the exact model and compatible state format.

## Required identity

The cloud prefiller and Tiercel must bind the snapshot to at least:

- exact model and quantisation fingerprint;
- tokenizer and chat-template fingerprint;
- canonical prompt token hash;
- RoPE and context configuration;
- KV types and relevant cache settings;
- llama.cpp state-format/build compatibility;
- architecture-specific state such as recurrent or hybrid checkpoints.

Approximate semantic equivalence is insufficient.

## Candidate modes

### Offline body foundry

1. Compile a stable Agent Body locally.
2. Send its canonical token sequence to a self-controlled cloud llama.cpp worker.
3. Prefill using the same GGUF and compatible runtime settings.
4. Export the slot-state binary.
5. Transfer the snapshot to Tiercel.
6. Validate its manifest and restore it.
7. Append local session context and decode locally.

This can tolerate a multi-gigabyte transfer because it occurs only when the body version changes.

### Live disaggregated prefill

Cloud computes KV for each request and streams it to Tiercel before local decode. Systems such as LMCache, NIXL and vLLM support this pattern inside high-bandwidth GPU clusters.

Across the public internet, transfer time for several gigabytes may exceed Tiercel's own prefill time. This mode is not the initial target.

### Commercial-provider cache export

Rejected for the current architecture. Ollama Cloud, OpenAI, Anthropic and similar APIs may cache prompts internally but do not provide a portable KV artifact that Tiercel can restore.

## Main risks

- llama.cpp slot files may not be portable across build versions or CUDA/Vulkan backends;
- the existing format has weak model-identity protection and known cross-quant limits;
- snapshots contain prompt tokens and sensitive derived state;
- Tiercel's speculative DFlash2 draft context is not necessarily included in existing slot persistence;
- transfer and storage can dominate for large Q8 KV states;
- a body update invalidates the changed segment and all following state;
- cloud prefill exposes the stable agent body to the cloud worker.

## Experiment ladder

Do not rent cloud hardware before the local slot experiment passes.

1. Restore a Tiercel-generated snapshot into the same process.
2. Restore it after a Tiercel service restart.
3. Restore it into a second compatible local process using the same model and build.
4. Test a deliberately different backend with the same build and model if available.
5. Only then run the exact GGUF and matched llama.cpp revision on a temporary cloud CUDA worker, export one synthetic snapshot and restore it on Tiercel.

At every stage compare:

- snapshot size and transfer time;
- restore time and appended-tail prefill;
- output correctness against a cold control;
- target and draft-model behaviour;
- cache-use metrics and failure mode for an incompatible manifest.

## Promotion criterion

Promote the offline body-foundry idea only if cross-machine restore is correct and total cloud prefill plus snapshot transfer is operationally worthwhile for an infrequently changing body. Do not promote live disaggregated prefill without evidence that network transfer beats local computation.

## Relevant upstream work

- llama.cpp issue 29201 proposes streaming slot-state export/import over HTTP for movement between backends.
- llama.cpp issue 29678 proposes session-keyed automatic save/restore and explicitly references cross-backend state movement.
- LMCache demonstrates disaggregated prefill using NIXL transports such as NVLink, RDMA or TCP, primarily within controlled inference clusters.
