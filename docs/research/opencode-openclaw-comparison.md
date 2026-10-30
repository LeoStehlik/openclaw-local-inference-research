# Hypothesis: OpenCode starts from a smaller or warmer sequence

## Question

Why can a local coding agent begin briskly and remain useful through a long session while an established OpenClaw agent arrives as a 150K-token cold request?

Having access to the same files does not prove the same content was injected into the request. The comparison must inspect request composition and cache use.

## Controlled experiment

Use:

- the same local model and quant;
- the same inference endpoint and server settings;
- the same workspace;
- the same synthetic task;
- fresh sessions;
- controlled cold and warm cache states.

Run OpenCode and OpenClaw separately. Do not run them concurrently against the single-slot endpoint.

## Capture

Record aggregate sizes and timings without storing raw private content:

- system and bootstrap tokens;
- agent-file tokens actually injected;
- memory tokens actually injected;
- tool-schema tokens;
- conversation and tool-result tokens;
- runtime metadata;
- total input tokens;
- cached and uncached prompt tokens;
- time to first token;
- output tokens and decode time.

Also record whether the client uses a stable session identifier, explicit cache settings or a persistent conversation process.

## Interpretations

### OpenCode sends much less cold context

Request assembly and eager hydration are the principal OpenClaw problem.

### Both send comparable cold context, OpenCode is faster

Investigate provider caching, slot affinity, prepared state or a different chat-template path.

### OpenCode is fast only after its first turn

Both systems depend on prefix reuse. The meaningful difference is the size and construction of the first request.

### OpenCode loses speed after eviction or restart

The apparent advantage came from a warm sequence rather than a durable cache architecture.

## Promotion gate

Do not redesign OpenClaw context assembly from assumptions. Promote an assembly change only after this comparison identifies the largest cold components and reproduces the difference under controlled cache state.
