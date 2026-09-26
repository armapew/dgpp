# Spark upstream history-control integration — 2026-09-26

Upstream [PR #54](https://github.com/HawkBearPig/dgpp/pull/54) was merged into the
maintained Spark branch and deployed on one GB10 with the NVIDIA
Qwen3.8-Flash-Next-NVFP4 checkpoint. The final source is
`f5739da0db709e1bdfd315ae6be7045da6e5e0f8`, tag `spark-2026.09.26.2`, release
`0.1.0+gf5739da0db70`. The preceding `0.1.0+gd44bebb61205` release and its
configuration remain available for rollback in the site operations workspace.

The change exposes native history controls, request-over-server defaults and
syntax-aware template capability detection. Default kwargs stay `{}`. The model,
four slots, 850048-token pool, MTP 2, 4096/1024 prefill budgets and 3 GiB prefix
snapshots are unchanged. No new history-discard policy was enabled globally.

## Validation

The merged candidate passed 77 service tests and 18 HTTP tests in a CPU-only
workstation build. Twelve renderings of the actual NVIDIA template matched both
the previous renderer and independent Jinja2 output, with thinking on/off and
multiple native history options. Default rendering and capability detection
were unchanged for this checkpoint.

The CUDA 13 release build then passed on the GB10. Native host CTest passed
26/27 groups and exposed a naming mistake in our fork's deployment example.
The example was renamed to `cluster_qwen-3.8-flash-next_nvfp4_w1_spark.example.json`
and added to the deployment catalogue; its contents are unchanged. The corrected
portability suite passed all 19 assertions, and portability/startup checks passed
again on GB10 after refreshing the release stamp. No engine/application/test
source changed between the initial host run and this filename/documentation fix.

## Live before/after sample

Both releases ran the same 19 requests: warmup, three decode classes, cold
2K/8K/32K prefill, eight fixed math questions and a four-request wave. All prompt
counts matched. All 15 sequential response strings matched exactly; math remained
7/8 with the same existing miss. Concurrent admission order differed and only
one of four concurrent texts matched, so this sample does not establish complete
concurrent bitwise parity. No request failed and token-accounting checks passed.

| Measurement | Previous | Updated |
| --- | ---: | ---: |
| Code decode, tok/s | 54.89 | 54.69 |
| JSON decode, tok/s | 61.07 | 60.78 |
| Agent-style decode, tok/s | 49.64 | 49.39 |
| 2K prefill, tok/s | 1618 | 1691 |
| 8K prefill, tok/s | 1758 | 1869 |
| 32K prefill, tok/s | 1642 | 1936 |
| Four 2K-input / 128-output requests, seconds | 14.00 | 13.91 |

This is a bounded regression sample with one observation per workload. The
prefill variation is not evidence of a new throughput optimization. Memory plan
remains 104.78 GiB; host MemAvailable at listening was 10.38 GiB.

Ten live history-option generations produced the expected status reply. Two
deliberately invalid controls were rejected with field-specific HTTP 400 errors.
The fixed history used 315 prompt tokens by default or with preservation true,
and 106 with preservation false. Doubling older reasoning raised the preserved
prompt to 519 tokens but left the dropped-history prompt at 106. Native keys,
ignored top-level fields, streaming and isolation between requests all behaved
as documented. The standard live API checker also passed stop, n=2, logit-bias,
cached-token and reasoning-token checks.

The release was packaged, installed, pinned and started with the ordinary cluster
command. The running executable and version matched the installed release.
The release pin was the only production JSON change; site environment was
preserved. No new GPU numerical kernels were introduced or independently
rebenchmarked as part of this integration.

Raw build/test logs, saved requests/responses, configuration identities and
rollback commands are retained in the separate Spark operations workspace under
`experiments/upstream-deployment-2026-09-26/` and its deployment runbook.
