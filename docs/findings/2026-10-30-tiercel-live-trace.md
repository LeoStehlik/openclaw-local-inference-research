# Tiercel live OpenClaw trace, 2026-10-30

## Environment

- OpenClaw 2026.10.6
- Qwen3.8-27B Q4
- llama.cpp with DFlash2 speculative decoding
- Ryzen AI Max+ 395
- 128GB unified memory
- 245,760-token model context
- Q8 KV
- prompt caching enabled
- idle-slot caching enabled
- 24GB RAM prompt cache
- one inference slot

## Cold request

- Input: 152,781 tokens
- Prompt processing: 545.252 seconds
- Prompt rate: 280.20 tokens/second
- Output: 321 tokens
- Decode time: 26.44 seconds
- Total: 571.693 seconds, approximately 9m32s

The server remained healthy and continuously busy. No timeout, deadlock or hardware fault explained the delay.

## Warm sequence

Ten cached follow-ups produced:

- median latency: 47.5 seconds;
- mean latency: 53.6 seconds;
- eight turns below one minute;
- all turns below two minutes;
- typical prefix similarity after hydration: 98 to 99 percent.

One 1,077-token intermediate output took 93.1 seconds despite fast cached prefill. Output length remains an independent latency term.

## Contention

A second unrelated request of roughly the same size did not match the hot prefix. With one inference slot it displaced the active sequence and another request waited behind it.

## Finding

The local model can run the established agent and becomes usable after prefix hydration. The immediate research problem is repeated cold computation, cache persistence and slot contention. Context capacity itself was sufficient.

Raw prompts, private memory and connection details are intentionally excluded.
