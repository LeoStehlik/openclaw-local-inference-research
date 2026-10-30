# Decision 0001: Preserve the agent body, change its loading lifecycle

## Status

Provisional research invariant.

## Decision

Do not make local inference viable by emptying identity, memory, rules, skills or tools. Preserve the OpenClaw agent model and investigate deterministic compilation, reusable KV state and selective paging of non-critical bulk context.

## Reason

The live trace showed that the full agent works correctly on the local model after prefill. Stable cached follow-ups were usable. The principal defect is repeated and poorly managed context computation, not inability of the model to hold the body.

## Consequences

- Frontier-provider defaults remain unchanged.
- Local experiments optimise uncached work and cache lifecycle.
- Hard rules and identity cannot depend on probabilistic retrieval.
- Soft memory and historical evidence may be paged when the capability impact is measured.
- A gateway or backend cache cannot independently decide the semantic structure of the agent body.
- A full router redesign is deferred until slot restore and deterministic-prefix experiments justify it.
