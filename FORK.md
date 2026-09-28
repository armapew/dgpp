# Spark maintenance fork

This fork serves `nvidia/Qwen3.8-Flash-Next-NVFP4` on a headless, single-node
NVIDIA GB10. Its main workload is concurrent agentic coding and tool use, often
with long contexts. Preserve correctness and accuracy when improving throughput.

## Accepted baseline

| Reference | Value |
| --- | --- |
| Maintained branch | `spark` |
| Immutable source tag | `spark-2026.09.28.2` |
| Exact deployed source | `eb5df5334114e0d9655dcb99d80c8fcf99286fc9` |
| Installed release | `0.1.0+geb5df5334114` |
| Upstream base | `743f5420438740c76a55791bd0fdf7e96ca24aab` |
| Previous fork history | `archive/spark-before-baseline-20260928` |

The tag fixes the exact tested build source. The maintained branch adds profile
documentation; its engine, launcher, tests and build files match the tag.
Previous baseline `spark-2026.09.28.1` / `d2db1a2` remains available for rollback.

Retained changes over the upstream base:

- Bound QSA scoring storage by the per-request context limit.
- Avoid empty QSA scoring blocks during prefill.
- Size serving logits storage for the runtime decode width.
- Reuse NVFP4 routed-expert gate/up and down weights across matching decode rows.
- Reuse QSA index keys across adjacent prefill queries and partition exact
  long-context decode selection, preserving scores and selected indices.
- Cache the existing FP8-to-BF16 dense conversion within a fixed memory budget.
- Batch ongoing and cached prefills on one Qwen node, retaining per-request
  state and the existing total token budget.
- Fix grouped-prefill draft hidden-row offsets and circular-buffer write races.

Against `d2db1a2`, measured cached C4 prefill time falls 27–40% on 20K/100K
histories. Isolating partitioned selection gives matching-response C1 decode
samples of 38.96 → 39.60 tok/s at 100K and 37.59 → 38.77 at 200K. Short-context
C1 shows no clear gain. The dense cache adds 4 GiB; observed host headroom is
about 4.2 GiB with the fixed startup guard unchanged.

Matched full tool-eval runs (92 cases, temperature zero, xhigh, seed 42,
concurrency 3, 65536 output tokens) score 167/184 versus 166/184: only TC-62
changes from pass to partial. Focused TC-61/62/63 replays pass with batching
both off and on. Concurrent long-context coding/tool checks pass 24/24 with
full prefix reuse. These results do not prove universal quality equivalence:
grouping changes execution shapes and can change generated text. Kernel exactness,
memory checks, scheduler tests and cache/MTP lifecycle checks also pass;
production remains MTP-1. Mixed prefill/decode was rejected after a numerical
screen, and unused-readout skipping is disabled because it showed no useful gain.

## Deployment profile

Use [the Spark example](deploy/cluster_qwen-3.8-flash-next_nvfp4_w1_spark.example.json).
It differs from the upstream NVIDIA w1 example only in `kv_capacity: 850048`
and `prefix_cache_gib: 3`. It retains four slots, BF16 KV, MTP depth 1,
4096/4096 prefill budgets, FP8 dense weights, the MMA head and full admission.
The model's per-request context limit is 262144 tokens including output.
Three full context reservations fit; four do not. Cached and active requests
share the KV pool, including output reservations.

Add [the runtime options](deploy/spark-runtime.env.example) to the site's
existing `.env`; do not replace its node/checkpoint settings. These options
select query tile 2, eight selection partitions, a 4096 MiB dense cache and
batched prefill. The dense cache retains exactly the BF16 values already
produced from FP8; it does not restore checkpoint precision. Set
`DGPP_BATCH_PREFILL=0` and restart to compare separate request prefills.

The example has no release pin. Add the exact installed release to the site's
deployment JSON, and keep its `.env`, addresses, credentials and local paths
outside the repository. The currently deployed binary remains pinned to the
release above; publishing source does not rebuild or restart it. An older-binary
rollback must also restore its original `.env`: older releases reject the new
node environment keys forwarded by the updated launcher.

## Ongoing development

Start each isolated improvement on a new `work/*` branch from `spark`. Keep
upstream integration separate from new optimizations. `origin` is
[armapew/dgpp](https://github.com/armapew/dgpp); `upstream` is
[HawkBearPig/dgpp](https://github.com/HawkBearPig/dgpp). `master` tracks the last
imported upstream revision and contains no fork patches.

Use focused checks appropriate to the change. For numerical or state changes,
check accuracy, prefix reuse, concurrency and relevant MTP depths. Keep
assistant-run evaluation brief unless a deeper campaign is authorized.
Use unchanged prompts/settings and temperature zero/xhigh for quality comparisons.
Do not run GPU tests on a serving instance handling other inference traffic.

After acceptance, merge into `spark`, tag the exact tested source and retain the
previous binary/configuration for rollback. Keep published tags fixed and use
non-forced pushes. Review incoming upstream changes on a candidate branch before
merging them, removing local patches when equivalent fixes become upstream.
