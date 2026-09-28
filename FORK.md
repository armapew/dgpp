# Spark maintenance fork

This fork serves `nvidia/Qwen3.8-Flash-Next-NVFP4` on a headless, single-node
NVIDIA GB10. Its workload is concurrent agentic coding and tool use with long
contexts. Correctness and accuracy take priority over throughput.

## Accepted baseline

| Reference | Value |
| --- | --- |
| Maintained branch | `spark` |
| Immutable source tag | `spark-2026.09.28.5` |
| Tested source | `0e2d46ed9dcd14bde6f77a1356fc23fe8a6fc313` |
| Installed release | `0.1.0+g0e2d46ed9dcd` |
| Imported upstream | `d027073e66f44f43e9c1580e14d9de4a43e13446` |
| Previous baseline | `spark-2026.09.28.4` / `2d7b0fc` |

The tag fixes the tested source. Maintained `spark` adds this profile document;
engine, launcher, tests and build files match the tag. Published tags are immutable.

The latest changes protect useful prefix snapshots, reduce n-gram table I/O,
and reuse index-key loads during decode. Upstream's 1024-token snapshot floor
and system-prompt snapshots are integrated with grouped prefill and cold-prefix
coalescing, including head-snapshot cleanup and waiting dependencies.

N-gram staging advises only the pages containing requested records, combines
repeated/adjacent page requests, and reuses copy workers for large chunks. Tiny
gathers remain allocation-free. Embedding bytes and their conversion are unchanged.
Decode query tile 2 preserves the score arithmetic and selections; its small
unbounded-position launch uses 128-pool stripes to retain sufficient parallelism.

Focused measurements on one GB10:

| Workload | Control | Selected |
| --- | --- | --- |
| 20K follow-up after 32 short requests, first token | 9.87 s | 0.27 s |
| New conversation with the same 20K system prompt, first token | 9.82 s | 0.22 s |
| 101K source-code prefill, first pass with table pages cold | 57.26 s | 56.41 s |
| Same source-code prefill, repeated | 56.83 s | 54.79 s |
| Table reads, first pass / repeat | 3.81 / 3.57 GiB | 1.95 / 0 GiB |

The staging comparison uses identical code and explicitly discards clean table
file-cache pages before each arm. Already-warm small working sets show essentially
unchanged prefill speed. The separate decode-key comparison gives C1 averages of
41.39 → 41.78 tokens/s at 20K and 39.12 → 39.34 at 100K, with matching responses.
Those are small screening differences; C2/C4 results are inconclusive and do not
establish a general throughput gain.

Concurrent 20K/100K retrieval, inspection and patch checks pass 24/24 at
temperature zero/xhigh; all 16 follow-ups reuse their long prefix. Native checks
cover exact n-gram bytes, concurrent gathers, QSA keys/selections/graph replay,
and identical resident/staged cache transcripts at MTP 1/2/5. Full model serving
checks use MTP 1. A full quality suite and 512K/YaRN quality are not established
by these checks. Scheduling and prefill boundaries can still change floating-point
execution and generated text.

The original BF16 QSA indexer is retained. The separate original-BF16 output-head
experiment is excluded: C1 decode fell about 11–13%, beyond the intended small
speed trade-off, without an established task-accuracy gain. The output head stays FP8.

Earlier retained work includes request-bounded QSA storage, empty scoring-block
avoidance, compact logits, NVFP4 expert reuse, exact dense-conversion caching,
partitioned selection, grouped prefill and draft-state fixes, and shared-prefix
coalescing. Mixed prefill/decode, grouped GDN projections and MoE worklists remain excluded.

## Deployment profile

Use [the Spark example](deploy/cluster_qwen-3.8-flash-next_nvfp4_w1_spark.example.json).
Its engine fields differ from the imported upstream NVIDIA w1 example only in
`kv_capacity: 850048`. The profile uses four slots, 262144 tokens per request
including output, BF16 KV, MTP-1, 4096/4096 prefill budgets, a 3 GiB prefix cache,
a 4 GiB dense-conversion cache, FP8 dense/MMA head, original BF16 indexing and
full admission. Three independent full-context reservations fit; four do not.

Append [the runtime options](deploy/spark-runtime.env.example) to the site's
existing `.env`. The new options are `DGPP_NGRAM_STAGING=1` and
`DGPP_QSA_DECODE_QUERY_TILE=2`; use 0 and 1 respectively for comparisons.
The memory plan remains 110.94 GiB with the fixed 4 GiB startup guard. Dense
caching preserves already-rounded FP8-to-BF16 values; it does not recover
checkpoint precision. Original BF16 indexer weights are loaded separately.

Pin the installed release in the deployment JSON. Keep site configuration,
addresses, credentials and local paths outside published source. Rollback must
restore both the previous JSON and `.env`; older parsers reject new node
settings even when their values are zero.

## Ongoing development

Start isolated `work/*` branches from `spark`. `origin` is
[armapew/dgpp](https://github.com/armapew/dgpp); `upstream` is
[HawkBearPig/dgpp](https://github.com/HawkBearPig/dgpp). `master` tracks the last
imported upstream revision. Review integrations separately and retain rollback
packages/configuration. Use non-forced pushes and keep release tags fixed.
Publishing source does not deploy it.

Evaluate long-context prefill, decode, cache reuse, concurrency and relevant MTP
depths with fixed prompts and temperature zero/xhigh. Keep checks focused; the
user runs broader benchmarks. Run one GPU workload at a time.
