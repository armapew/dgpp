# Spark maintenance fork

This fork serves `nvidia/Qwen3.8-Flash-Next-NVFP4` on one headless NVIDIA GB10.
Its workload is concurrent agentic coding and tool use with long contexts.
Correctness and accuracy take priority over throughput.

## Current release

| Reference | Value |
| --- | --- |
| Maintained branch | `spark` |
| Immutable source tag | `spark-2026.09.29.2` |
| Tested source | `f380d2704d9131ed6f7f752db495577c958dc353` |
| Installed release | `0.1.0+gf380d2704d91` |
| Imported upstream | `84028a497443782d9f3dde08895c638658115933` |
| Previous baseline | `spark-2026.09.29.1` / `6205065` |

The tag fixes the tested source. Maintained `spark` adds this profile document;
engine, launcher, tests and build files match the tag. Published tags are immutable.

`DGPP_MOE_GU_QUANT=1` fuses calibrated routed expert gate/up projection,
SwiGLU and activation quantization during prefill. It preserves the existing
BF16 roundings and accumulation order and reuses activation scratch. Set it to
0 and restart to restore the original chain. Restored resident images recover
the calibration metadata needed to select the same path as checkpoint loading.

Matched 93.8K code prefill averaged **1873 → 1937 tok/s** over two runs;
100K inventory prefill improved **1864 → 1931 tok/s**. The observed gain is
**3.4–3.6%**. Cached 100K aggregate decode at C1/C2/C4 remained essentially
flat: **42.1/60.9/80.2 → 42.2/61.5/80.0 tok/s**. These are focused comparisons,
not full benchmark-suite results.

Upstream snapshot-boundary filtering is retained. Upstream removed the old cold
`engine.prefill_group` switch after GLM consistency work. Qwen's grouped chunk
path remains enabled and is still batch-dependent; no batch-invariance claim
extends to this model.

Offline GEMM choices, newer cuBLASLt, prefetch suppression and small-group
expert decode variants showed no useful serving gain and are excluded. FP32
expert-down intermediates cost about 6% prefill throughput without a demonstrated
task-quality benefit. Experimental source remains separate from this release.

## Deployment profile

Use [the Spark example](deploy/cluster_qwen-3.8-flash-next_nvfp4_w1_spark.example.json)
and append [the runtime options](deploy/spark-runtime.env.example) to the existing
site `.env`. The profile retains the 850048-token BF16 KV pool, 262144 tokens per
request including output, four slots, MTP-1, 4096/4096 prefill budgets, a 3 GiB
prefix cache, a 4 GiB dense-conversion cache, FP8 dense/MMA head, original BF16
indexer and target K/V projections, draft shortlist and bounded n-gram lookahead.
Three full-context reservations fit; four do not.

`DGPP_PREFILL_LAYER_YIELD=8` retains the earlier responsiveness improvement:
active generations get a decode pass at complete prefill layer boundaries.
The existing BF16 hyper state and 4096-token prefill shape are preserved. It
requires one resident GPU and full reservation and uses **80 MiB** of workspace.
Set it to 0 and restart to disable yielding. Existing generations progress sooner;
new prompts can wait longer for their first token. C1 has no peer to yield to.

The memory plan is **111.05 GiB + 4 GiB guard**. Earlier accepted attention,
logits, expert-reuse, dense-cache and shared-prefix improvements remain. No
checkpoint, KV-cache precision, pool capacity or sampling change is made.

Concurrent 20K/100K retrieval, inspection and patch checks pass **24/24**, with
all 16 follow-ups reusing their prefixes. Fixed-history cold/cached and C1/C4
probes match captured layer values and final logits bitwise with fusion off/on.
Native checks cover ragged shapes, graph replay, resident restoration, cache/slot
reuse, MTP 1–5 and yielding; CUDA memcheck reports zero errors. Full-model checks
use MTP-1. These checks do not establish full tool-eval quality, batch invariance,
512K/YaRN quality or a fix for historical loops.

Pin the installed release. Keep site configuration and credentials outside
published source. Rollback must restore both the previous JSON and `.env`;
older parsers reject new node settings even when their values are zero.

## Ongoing development

Start `work/*` branches from `spark`. `origin` is [armapew/dgpp](https://github.com/armapew/dgpp);
`upstream` is [HawkBearPig/dgpp](https://github.com/HawkBearPig/dgpp). `master` tracks
the last imported upstream revision. Review integrations separately, retain
rollback packages/configuration, and use non-forced pushes. Publishing source
and deploying it are separate operations.

Evaluate long-context prefill/decode, cached follow-ups, concurrency and relevant
MTP depths with fixed prompts and temperature zero/xhigh. Keep routine checks
focused; the user runs broader benchmarks. Run one GPU workload at a time.
