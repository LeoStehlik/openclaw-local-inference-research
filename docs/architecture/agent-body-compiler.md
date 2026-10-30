# Agent Body compiler

## Purpose

Compile OpenClaw's durable agent context into a deterministic, versioned prefix that a capable inference backend can prefill once and reuse safely.

This does not replace Markdown sources, memory retrieval, tools or the existing agent philosophy. It changes how stable context is assembled, identified and loaded.

## The expensive artifact

There are three distinct artifacts:

1. **Canonical text**: deterministic rendering of the agent body.
2. **Token sequence**: canonical text processed by the model tokenizer.
3. **KV state**: the token sequence evaluated through the model.

Persisting text or token IDs avoids only cheap parsing and tokenisation. Reusing KV state avoids the expensive transformer prefill.

## Inputs

Candidate inputs are divided by lifecycle rather than file type.

### Stable body

- system authority and hard rules;
- identity and behavioural contract;
- stable tool catalogue and schemas;
- compact capability, memory and infrastructure route maps.

### Versioned body

- durable memory summary;
- current operator preferences;
- slowly changing project or environment capsule.

### Per-session tail

- relevant retrieved memory pages;
- channel and sender context;
- recent conversation;
- current runtime facts and user request.

Raw historical receipts, old transcripts and unrelated memory do not belong in the stable body merely because they exist.

## Compiler pipeline

### 1. Source discovery

Resolve the exact OpenClaw sources that would otherwise enter the prompt. Record source identity and scope without copying private content into diagnostics.

### 2. Deterministic normalisation

Normalise only representation details whose removal cannot change meaning:

- line endings;
- known generated wrappers;
- stable ordering of unordered metadata;
- canonical JSON and tool-schema serialization.

Do not use approximate semantic equivalence as authority for rules or identity.

### 3. Segmentation

Render ordered segments from least volatile to most volatile. Every segment declares its dependencies and invalidation class.

Example:

```text
A runtime contract
B identity and hard rules
C stable tools
D durable memory map
E current-state capsule
F retrieved task context
G session history and current turn
```

A change to D invalidates D onward. A through C remain reusable.

### 4. Canonical rendering

Produce byte-stable messages using an explicit compiler version and chat template. Dynamic timestamps, request IDs and current-turn metadata must occur after stable segments.

### 5. Tokenisation and manifest

Tokenise with the target model tokenizer and produce a manifest containing:

```text
compiler version
model fingerprint
tokenizer fingerprint
chat-template fingerprint
ordered segment hashes
token counts and boundaries
full prefix token hash
```

The full cache identity binds all of these values.

### 6. Materialisation

Ask a provider adapter to prefill the canonical prefix and return a provider-specific cache identity or snapshot reference. Ordinary providers may ignore this phase and receive the normal request.

### 7. Session assembly

Start a session from the prepared body state, then append retrieved context, recent history and the current turn. The logical prompt remains complete even when the backend avoids recomputing its prefix.

## Update lifecycle

Body updates are generational:

1. detect a source change;
2. rebuild affected segments and hashes;
3. materialise the new body in the background;
4. verify it;
5. atomically make it the default for new sessions;
6. allow existing sessions to finish on the old generation or migrate at a safe boundary.

Do not mutate the middle of a live KV state. Later token state depends on all earlier tokens.

## Optional System 1 assistance

A small classifier can label source changes as cosmetic, duplicate, new fact, changed fact or instruction change. It can help consolidate soft memory into structured facts.

It cannot authorise KV reuse for two different token sequences. Exact token identity remains the final cache-validity test.

## Provider boundary

The generic compiler should expose capabilities such as:

```text
preparePrefix(manifest, messages)
restorePrefix(cacheIdentity)
appendSessionTail(messages)
invalidatePrefix(cacheIdentity)
observeCacheUse()
```

The first Tiercel adapter may translate these into llama.cpp slot save, restore and prompt-cache operations. Those details must not leak into generic frontier-provider behaviour.

## Material design alternatives

### OpenClaw-native compiler plus provider adapter

Best long-term fit. OpenClaw knows the semantic lifecycle of its context, while the adapter knows backend cache mechanics.

### Gateway-only interception

Useful for proving slot restore quickly. Limited because a gateway sees rendered prompts but does not know which OpenClaw segments are authoritative or safe to consolidate.

### Inference-only automatic caching

Already helps exact repeated prefixes. Insufficient by itself because it cannot protect an agent body, control prompt ordering or distinguish valuable state from disposable sessions.

## Recommended research direction

Persistent Qwen3.8 KV restore is now proven on Tiercel at 9K scale. The next step is deterministic compilation for one disposable agent fixture, followed by the 100K-plus restore benchmark. Do not build a general KV lifecycle manager before those two facts are established.
