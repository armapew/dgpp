# Spark maintenance fork

This fork serves `nvidia/Qwen3.8-Flash-Next-NVFP4` on one headless NVIDIA GB10.
Its workload is concurrent agentic coding and tool use with long contexts.
Correctness and accuracy take priority over throughput.

## Current release

| Reference | Value |
| --- | --- |
| Maintained branch | `spark` |
| Immutable source tag | `spark-2026.09.29.1` |
| Tested source | `620506561ee87155ac00fc0c8e575e70a915c907` |
| Installed release | `0.1.0+g620506561ee8` |
| Imported upstream | `84028a497443782d9f3dde08895c638658115933` |
| Previous baseline | `spark-2026.09.28.7` / `44c5777` |

The tag fixes the tested source. Maintained `spark` adds this profile document;
engine, launcher, tests and build files match the tag. Published tags are immutable.

The profile now keeps target-model attention K/V projections in original
checkpoint BF16. `DGPP_QSA_TARGET_KV_BF16=1` enables it; 0 restores their FP8
conversion on restart. Q/O, output-head and draft precision stay unchanged.
Resident images have separate precision identities. The added raw weights are
about **30 MiB**, with no KV token-pool reduction.

Matched 93.8K code prefill averaged **1864 → 1871 tok/s**, effectively flat.
C1 decode changed with text and draft acceptance (about −1.7% at 20K and +4.8%
at 100K), so no kernel speedup or task-quality gain is claimed. This removes an
extra checkpoint-weight rounding; the full task benchmark remains the quality gate.

Upstream snapshot-boundary filtering is retained. Upstream removed the old cold
`engine.prefill_group` switch after GLM consistency work. Qwen's grouped chunk
path remains enabled and is still batch-dependent; no batch-invariance claim
extends to this model.

Shared-expert conversion caching, fused exact QSA selection, cross-slot prefix
score reads, and GDN/residual preparation fusions are excluded. Focused comparisons
found slower execution or no useful full-model gain. Their experimental source
is separate from this release.

## Deployment profile

Use [the Spark example](deploy/cluster_qwen-3.8-flash-next_nvfp4_w1_spark.example.json)
and append [the runtime options](deploy/spark-runtime.env.example) to the existing
site `.env`. The profile retains the 850048-token BF16 KV pool, 262144 tokens per
request including output, four slots, MTP-1, 4096/4096 prefill budgets, a 3 GiB
prefix cache, a 4 GiB dense-conversion cache, FP8 dense/MMA head, original BF16
indexer, draft shortlist and bounded n-gram lookahead. Three full-context
reservations fit; four do not.

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
all 16 follow-ups reusing their prefixes. Native checks cover checkpoint bytes,
resident images, cache/graph/MTP 1/2/5, yielding and positional ceilings; CUDA
memcheck reports zero errors. Full-model checks use MTP-1. These checks do not
establish full tool-eval quality, 512K/YaRN quality or a fix for historical loops.

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
