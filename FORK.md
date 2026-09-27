# Spark maintenance fork

This fork serves `nvidia/Qwen3.8-Flash-Next-NVFP4` on a headless, single-node
NVIDIA GB10. Its main workload is concurrent agentic coding and tool use, often
with long contexts. Preserve correctness and accuracy when improving throughput.

## Accepted baseline

| Reference | Value |
| --- | --- |
| Maintained branch | `spark` |
| Immutable source tag | `spark-2026.09.28.1` |
| Exact deployed source | `d2db1a2232fcae2308f5b81c11df1a71d06526a0` |
| Installed release | `0.1.0+gd2db1a2232fc` |
| Upstream base | `743f5420438740c76a55791bd0fdf7e96ca24aab` |
| Previous fork history | `archive/spark-before-baseline-20260928` |

The tag fixes the exact accepted build source. The maintained branch adds this
documentation and a deployment example; its engine, tests and build files match
the tag. The previous fork history and older release tags remain available.

Retained changes over the upstream base:

- Bound QSA scoring storage by the per-request context limit.
- Avoid empty QSA scoring blocks during prefill.
- Size serving logits storage for the runtime decode width.
- Reuse NVFP4 routed-expert gate/up and down weights across matching decode rows.

The down-projection change passed bitwise kernel, graph and memory checks, plus
cache/slot lifecycle fixtures at MTP depths 1–5. Repeated C4 benchmarks with
2048 prompt and 2048 output tokens measured mean aggregate decode of
78.3 → 79.9 tok/s (+2.04%). Prefill was effectively unchanged and reported total
time did not improve. The shorter cached-20K screen found no speed gain; these
results do not establish a long-context improvement or a new quality score.

## Deployment profile

Use [the Spark example](deploy/cluster_qwen-3.8-flash-next_nvfp4_w1_spark.example.json).
It differs from the upstream NVIDIA w1 example only in `kv_capacity: 850048`
and `prefix_cache_gib: 3`. It retains four slots, BF16 KV, MTP depth 1,
4096/4096 prefill budgets, FP8 dense weights, the MMA head and full admission.
The model's per-request context limit is 262144 tokens including output.
Three full context reservations fit; four do not. Cached and active requests
share the KV pool, including output reservations.

The example has no release pin. Add the exact installed release to the site's
deployment JSON, and keep its `.env`, addresses, credentials and local paths
outside the repository. The currently deployed binary remains pinned to the
release above; publishing source does not rebuild or restart it.

## Ongoing development

Start each isolated improvement on a new `work/*` branch from `spark`. Keep
upstream integration separate from new optimizations. `origin` is
[armapew/dgpp](https://github.com/armapew/dgpp); `upstream` is
[HawkBearPig/dgpp](https://github.com/HawkBearPig/dgpp). `master` tracks the last
imported upstream revision and contains no fork patches.

Use focused checks appropriate to the change. For numerical or state changes,
check accuracy, prefix reuse, concurrency and relevant MTP depths. Keep
assistant-run evaluation brief; the operator runs the full benchmark suites.
Use unchanged prompts/settings and temperature zero/xhigh for quality comparisons.
Do not run GPU tests on a serving instance handling other inference traffic.

After acceptance, merge into `spark`, tag the exact tested source and retain the
previous binary/configuration for rollback. Keep published tags fixed and use
non-forced pushes. Review incoming upstream changes on a candidate branch before
merging them, removing local patches when equivalent fixes become upstream.
