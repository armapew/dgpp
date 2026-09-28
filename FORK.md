# Spark maintenance fork

This fork serves `nvidia/Qwen3.8-Flash-Next-NVFP4` on a headless, single-node
NVIDIA GB10. Its workload is concurrent agentic coding and tool use with long
contexts. Correctness and accuracy take priority over throughput.

## Current release

| Reference | Value |
| --- | --- |
| Maintained branch | `spark` |
| Immutable source tag | `spark-2026.09.28.6` |
| Tested source | `f6a0214a8484771feff0211dbfc93f295b645663` |
| Installed release | `0.1.0+gf6a0214a8484` |
| Imported upstream | `4564724b4c820ac8d4ee33f584c94b1855f99a53` |
| Previous baseline | `spark-2026.09.28.5` / `0e2d46e` |

The tag fixes the tested source. Maintained `spark` adds this profile document;
engine, launcher, tests and build files match the tag. Published tags are immutable.

Upstream positional-ceiling guards reject oversized prompt/output reservations
and pad speculative verification rows at the boundary. Our existing attention
workspace optimization is retained alongside its upstream merge.

The MTP draft head scores the first 65536 vocabulary rows and the final 128
rows. Target generation and verification retain the full vocabulary. This saves
draft-head work without changing target weights or cache precision, although
proposal acceptance and the performance benefit depend on the workload.

Bounded n-gram lookahead computes upcoming prompt hashes on one host worker and
advises the required file pages ahead of the GPU walk. Jobs own their token
copies; queued work is cancelled on slot reset. The existing embedding gather
remains authoritative. No extra GPU allocation or KV format change is introduced.

Focused comparisons on one GB10, with each option isolated:

| Workload | Control | Selected |
| --- | --- | --- |
| C1 decode, 20K context | 41.53 tok/s | 42.43 tok/s (+2.2%) |
| C1 decode, 100K context | 39.18 tok/s | 40.12 tok/s (+2.4%) |
| Cold 104K code prefill, first pair | 1801 tok/s | 1837 tok/s |
| Cold 104K code prefill, repeat pair | 1782 tok/s | 1836 tok/s |
| Warm repeat of the same code prefill | 1846 tok/s | 1841 tok/s |

Decode values average two 512-token runs per setting; all four matched C1
responses are identical. Cold comparisons discard only clean n-gram file-cache
pages with the server stopped. C4 decode and warm prefill are essentially
unchanged. These focused measurements do not establish a universal gain.

The combined release passes 24/24 concurrent 20K/100K retrieval, inspection and
patch checks at temperature zero/xhigh, with all 16 follow-ups reusing their long
prefix. Native checks cover selected draft logits, graph replay, cold/cached
slot reuse and MTP 1/2/5, plus upstream HTTP and positional-boundary guards.
Full model checks use MTP 1. A full quality suite, statistical sampled-output
equivalence and 512K/YaRN quality are not established by these checks. Scheduling
and execution shapes can still change floating-point results and generated text.

A CUTLASS SM121 expert prototype produced identical tested outputs, but converting
the existing weight scales made complete operations 10–27% slower. It is excluded;
caching converted down-projection scales alone would need about 2.4 GiB.

Earlier retained work includes the original BF16 QSA indexer, prefix retention
and system snapshots, exact n-gram staging, decode key reuse, request-bounded QSA
storage, empty scoring-block avoidance, compact logits, NVFP4 expert reuse, dense
conversion caching, partitioned selection, grouped prefill/draft-state fixes and
shared-prefix coalescing. Mixed prefill/decode, grouped GDN projections, MoE
worklists and the slower original-BF16 output head remain excluded.

## Deployment profile

Use [the Spark example](deploy/cluster_qwen-3.8-flash-next_nvfp4_w1_spark.example.json).
Its engine fields differ from the imported upstream NVIDIA w1 example only in
`kv_capacity: 850048`. The profile uses four slots, 262144 tokens per request
including output, BF16 KV, MTP-1, 4096/4096 prefill budgets, a 3 GiB prefix cache,
a 4 GiB dense-conversion cache, FP8 dense/MMA head, original BF16 indexing and
full admission. Three independent full-context reservations fit; four do not.

Append [the runtime options](deploy/spark-runtime.env.example) to the site's
existing `.env`. The new options are `DGPP_DRAFT_VOCAB_LIMIT=65536` and
`DGPP_NGRAM_LOOKAHEAD_TOKENS=8192`; either can be disabled with 0 and a restart.
The memory plan remains 110.94 GiB with the fixed 4 GiB startup guard. Lookahead
uses bounded host metadata and the existing file cache. Dense caching preserves
already-rounded FP8-to-BF16 values; original BF16 indexer weights are separate.

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
