# Tiercel slot-checkpoint backport

## Finding

Qwen3.8 slot save and restore on the original Tiercel runtime was cosmetic. The API reported a successful restore, but the next request had no usable recurrent context checkpoints and reprocessed the complete prompt.

Backporting merged llama.cpp PR [#26004](https://github.com/ggml-org/llama.cpp/pull/26004) to the Strix Halo runtime restored those checkpoints correctly.

## Controlled result

The fixture used Qwen3.8-27B Q4, the Q4 DFlash2 drafter, Q8 target KV, one 245,760-token slot and a deterministic prompt spanning multiple ubatches. RAM and idle-slot caches were disabled so they could not hide a failed disk restore.

- Cold path: 9,110 prompt tokens, about 21 seconds wall time.
- Stock restored path: zero cached tokens and the whole prompt reprocessed.
- Fixed restored path: 7,062 cached tokens and a 2,050-token checkpoint tail.
- Fixed restored wall time: about 5.1 seconds.
- The same reuse survived a complete server restart.
- Generated output matched the live path.
- DFlash2 drafted two tokens and accepted both.

The fixed snapshot was about 872 MB. Roughly 379 MB was the recurrent target, draft and speculative checkpoint appendix. Long-context snapshot size and restore latency still need direct measurement.

## Implication

The backend can now prefill a stable Agent Body once, save it, survive model process loss and resume from the latest valid recurrent checkpoint. This does not make different prefixes equivalent. OpenClaw still needs deterministic prefix assembly so new sessions share the saved leading tokens.

## Sources

- [Tiercel Qwen3.8 + DFlash2 implementation and benchmarks](https://github.com/LeoStehlik/qwen3.8-27b-strix-halo-dflash2)
- [Detailed controlled result](../research/tiercel-slot-checkpoint-backport-results.md)
- [Upstream llama.cpp checkpoint persistence PR](https://github.com/ggml-org/llama.cpp/pull/26004)
