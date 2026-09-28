# Spark maintenance fork

This fork serves `nvidia/Qwen3.8-Flash-Next-NVFP4` on one headless NVIDIA GB10.
Its workload is concurrent agentic coding and tool use with long contexts.
Correctness and accuracy take priority over throughput.

## Current release

| Reference | Value |
| --- | --- |
| Maintained branch | `spark` |
| Immutable source tag | `spark-2026.09.28.7` |
| Tested source | `44c5777f9e25ee7914bab3a7d902a79e445830da` |
| Installed release | `0.1.0+g44c5777f9e25` |
| Imported upstream | `fb4d2ad63a17db828115091a833f7fa917906ae1` |
| Previous baseline | `spark-2026.09.28.6` / `f6a0214` |

The tag fixes the tested source. Maintained `spark` adds this profile document;
engine, launcher, tests and build files match the tag. Published tags are immutable.

Upstream now defaults `engine.prefill_group` off for cold group admission.
Our grouped continuation/chunk path is separate and remains enabled. This does
not establish batch invariance for the whole engine.

Prefill can yield to active generations at complete layer boundaries. It saves
and restores the existing BF16 hyper state, preserving the 4096-token prefill
shape and the separate decode path. The Spark profile uses
`DGPP_PREFILL_LAYER_YIELD=8`; 0 disables it. This requires one resident GPU and
full reservation. Cancellation retains the normal scheduler retirement boundary.
The extra workspace is **80 MiB** at the selected shape.

Matched temperature-zero/xhigh checks, with a cached 20K parent and a newly
arriving 100K prompt:

| Measurement | Yielding off | Eight-layer interval |
| --- | --- | --- |
| Longest parent pause, 1024 generated tokens | 2.51 s | 0.54 s |
| Completion of a 256-token parent response | 60.1 s | 45.7 s |
| Combined runtime, 1024-token case | 78.1 s | 78.4 s |
| New prompt's first token, 1024-token case | 54.8 s | 61.0 s |

Both responses match exactly in each comparison. Token budgets include reasoning.
Yielding improves responsiveness under overlapping work; the new prompt waits
longer while the existing generation makes progress. C1 has no peer to yield to.

Concurrent 20K/100K retrieval, inspection and patch checks pass **24/24**, with
all 16 follow-ups reusing their long prefix. Native checks cover unchanged logits,
graph/cache/MTP 1/2/5, stop/cancel handling and positional boundaries; CUDA
memcheck reports zero errors. These focused checks do not replace the user's
full benchmark suite or establish full 512K/YaRN quality. Historical reasoning
loops remain unresolved, and scheduling can still change floating-point execution
shapes and generated text.

Prompt lookup, adaptive draft-chain trimming, embedding-record caching and the
shared expert scale-layout/CUTLASS prototype are excluded. Their focused tests
found overhead or no dependable full-model gain. The expert prototype preserved
tested values and improved its isolated operation, but model throughput stayed
within roughly 1% of the control. There is no new CUTLASS build dependency.

## Deployment profile

Use [the Spark example](deploy/cluster_qwen-3.8-flash-next_nvfp4_w1_spark.example.json)
and append [the runtime options](deploy/spark-runtime.env.example) to the existing
site `.env`. The profile retains the 850048-token BF16 KV pool, 262144 tokens per
request including output, four slots, MTP-1, 4096/4096 prefill budgets, a 3 GiB
prefix cache, a 4 GiB dense-conversion cache, FP8 dense/MMA head, original BF16
indexer, draft shortlist and bounded n-gram lookahead. Three full-context
reservations fit; four do not.

The startup memory plan is **111.02 GiB + 4 GiB guard**. Previous accepted
attention, logits, expert-reuse, dense-cache and shared-prefix improvements remain.
No checkpoint, KV precision, pool capacity or sampling change is made.

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
