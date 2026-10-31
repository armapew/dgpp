# DeepSeek-V4.1-Flash-1017 EXL3 2.8 bpw candidate

This candidate targets
[`jayleaton/DeepSeek-V4.1-Flash-1017-EXL3-2.8bpw`](https://huggingface.co/jayleaton/DeepSeek-V4.1-Flash-1017-EXL3-2.8bpw)
at revision `859396591c340195526aaf61f94b4ad5323a5e18` on two GB10 nodes.
The initial scope is text inference. DeepSeek vision and Responses/client
integration are deferred until the text baseline has been evaluated.

## Weight contract

The checkpoint has no `exl3_k_map.json`. The config reader discovers widths
from safetensors headers; the binding table still supplies and checks every
architectural shape. Gate/up widths agree within each expert, while down
projections can be wider, and experts within a layer can differ. The seven
backbone gate/up/down patterns are 2/2/2, 2/2/3, 3/3/3, 3/3/4, 4/4/4,
4/4/5 and 5/5/5. Shared experts use 5 or 6 bits. Dense projections carry
their own widths, including 8-bit indexer keys and the 6-bit output head.

The native BF16 routers/heads and FP32 biases retain their stored precision.
The other EXL3 pack's F16/F32 native representations remain accepted only
through the existing exact-conversion checks. Eleven extra BF16 copies of
compressor/indexer matrices are validated and skipped when their EXL3 group
is present, matching the pack's TensorFold selection. No requantization or
expert pruning is part of this port.

The CUDA core includes 2/5-bit forms. Heterogeneous MoE launches read each
selected expert's width from its view; all threads in a CTA take the same
branch into the existing per-width arithmetic. Uniform packs retain their
specialized width-pair paths. Prefill, decode, graph tables and TP slices
must all pass the mixed-width fixtures before serving this candidate.

## Initial profile

The [two-node template](../../deploy/cluster_deepseek-v4.1-flash-1017_exl3-2.8bpw_w2.example.json)
uses four slots, a 1,048,576-token shared pool, 2,048-row prefill budgets,
a 1 GiB prefix arena and fixed depth-5 DSpark verification. It deliberately
does not reuse the four-node 3.5 bpw template's measured scheduling costs.
This is a bring-up allocation, pending load and concurrent-prefill peak checks.

DeepSeek's existing DGPP cache formats remain FP4-block main compressed KV,
FP8-block sliding windows and FP8 index keys. These differ from TensorFold's
default FP8 split main cache; the same weight pack does not imply identical
cache arithmetic or output. Keep that distinction in evaluation records.

The original Engram tables are external, selected by `engine.engram_source`
at revision `1bae0e1088993b9b82de08f59de5be5a34e42170`. Only the table tensors
from shards 47/48 are admitted. Generate their hash sidecar with
`tools/dsv41_engram_tables.py` against the served tokenizer. The sidecar's
token map and multipliers must match the reference.

Bind the private API to loopback and retain the site's authenticated proxy.
Temperature-zero comparisons use seed 42 and `reasoning_effort: "xhigh"`.
Include sampled traffic, cached follow-ups and cancellation in focused checks.
Full quality benchmarks belong to the operator after the candidate is ready.
