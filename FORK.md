# Spark maintenance fork

This fork serves `nvidia/Qwen3.8-Flash-Next-NVFP4` on a headless, single-node
NVIDIA GB10. Its main workload is concurrent agentic coding and tool use with
long contexts. Correctness and accuracy take priority over throughput.

## Accepted baseline

| Reference | Value |
| --- | --- |
| Maintained branch | `spark` |
| Immutable source tag | `spark-2026.09.28.4` |
| Exact deployed source | `2d7b0fc3b1413cfc257ea9c6f9fe57f171a366cc` |
| Installed release | `0.1.0+g2d7b0fc3b141` |
| Upstream base | `743f5420438740c76a55791bd0fdf7e96ca24aab` |
| Previous baseline | `spark-2026.09.28.3` / `8e1bfb7` |

The tag identifies the tested build source. The maintained branch adds this
profile documentation; engine, launcher, tests and build files match the tag.
Published tags remain immutable.

The latest change preserves the QSA indexer's projection weights in their original
checkpoint BF16 precision, including the MTP layer. Other dense projections stay
FP8 and expert weights retain their checkpoint formats. This removes one extra
quantization step from attention selection; a task-accuracy gain is not established.
The memory plan increases by about 20 MiB. Resident weight images use separate
identities for FP8 and BF16 indexer weights.

Focused validation passes exact checkpoint-weight loading, separate resident
image restoration, dense-cache equivalence and four-slot cache/graph lifecycle
checks at MTP 1/2/5. Concurrent 20K/100K retrieval and coding/tool checks pass
24/24 at temperature zero/xhigh, with prefix hits on every follow-up. No full
quality suite or matched throughput comparison was run for this release.

The preceding change coalesces simultaneous requests with an identical cold prefix.
Followers wait for an older executing request's reusable snapshot, then compute
their own suffixes. An untaken body snapshot can move to a deeper common existing
block boundary. MTP lookahead is part of the match, waiting is bounded, unrelated
work can proceed, and cancellation releases the dependency.

Measured median time to first token for four simultaneous shared histories:

| History | Previous baseline | Coalescing | Time reduction |
| --- | --- | --- | --- |
| 20K | 41.74 s | 16.96 s | 59% |
| 100K | 224.00 s | 58.89 s | 74% |

These are matched synthetic shared-prefix samples, not general kernel speedups.
Already-cached histories and prompts that differ near the beginning have less
to gain. The leader can pause for roughly 2–3 seconds while followers prefill
under the existing 4096-token busy budget. Existing snapshot storage and the
memory budget are retained.
The preceding baseline's full tool-eval runs (92 cases, temperature zero, xhigh, seed 42, concurrency 3,
65536 output tokens) score **163/184 control versus
164/184 selected profile**. Four outcomes improve and four worsen; 84 match.
Concurrent 20K/100K retrieval, inspection and patch checks pass 24/24. Scheduler, configuration and cache/MTP
1/2/5 checks pass. Generated text can vary with scheduling; these samples do not
prove universal quality equivalence or resolve the historical reasoning loops.

Earlier retained changes include request-bounded QSA storage, empty scoring-block
avoidance, compact logits, NVFP4 gate/up/down reuse, QSA query reuse and exact
partitioned selection, exact dense-conversion caching, grouped ongoing/cached
prefills and grouped draft-state fixes. Grouped GDN projections and MoE worklists
remain excluded. The earlier BF16 indexer quality run was incomplete; its loop
does not establish causality, since previous official builds also looped.

## Deployment profile

Use [the Spark example](deploy/cluster_qwen-3.8-flash-next_nvfp4_w1_spark.example.json).
It differs from the upstream NVIDIA w1 example only in `kv_capacity: 850048`
and `prefix_cache_gib: 3`. It retains four slots, BF16 KV, MTP-1, 4096/4096
prefill budgets, FP8 dense weights, the MMA head and full admission. Per-request
context is 262144 tokens including output. Three independent full reservations
fit; four do not. Cached and active requests share the KV pool.

Append [the runtime options](deploy/spark-runtime.env.example) to the site's
existing `.env`. They select query tile 2, eight selection partitions, a 4096 MiB
dense cache, grouped ongoing/cached prefill, cold-prefix coalescing and
`DGPP_QSA_INDEXER_BF16=1`. The memory plan is 110.94 GiB with the fixed 4 GiB startup guard. The dense cache
preserves the existing FP8-to-BF16 conversion; it does not restore checkpoint
precision; original BF16 indexer weights are loaded separately. Set
`DGPP_QSA_INDEXER_BF16=0` and restart for an FP8-indexer comparison on the same
binary. Set `DGPP_PREFIX_COALESCE=0` to disable cold-prefix coalescing.

Pin the exact installed release in the site's deployment JSON. Keep `.env`,
addresses, credentials and local paths outside published source. Publishing
source does not rebuild or restart the deployment. Rollback to an older binary
must restore its original JSON and `.env`: older releases reject the new node
environment key, even with value zero.

## Ongoing development

Start isolated changes on `work/*` branches from `spark`. Keep upstream
integration separate. `origin` is [armapew/dgpp](https://github.com/armapew/dgpp);
`upstream` is [HawkBearPig/dgpp](https://github.com/HawkBearPig/dgpp). `master`
tracks the last imported upstream revision without fork patches.

Check accuracy, prefix reuse, concurrency and relevant MTP depths for numerical
or state changes. Keep validation focused unless a deeper campaign is authorized.
Use unchanged prompts/settings and temperature zero/xhigh for comparisons. Run
one GPU workload at a time. After acceptance, merge into `spark`, tag the tested
source and retain the previous binary/configuration. Use non-forced pushes and
keep release tags fixed. Remove equivalent local patches during upstream review.
