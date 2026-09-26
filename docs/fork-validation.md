# Validating Spark fork changes

Use the currently deployed release as the comparison baseline, with the same
checkpoint, prompts, sampling settings and output budgets. Record the source
revision, binary checksum, effective configuration and hardware state for each
candidate. See [CONTRIBUTING.md](../CONTRIBUTING.md) and [testing.md](testing.md)
for build dependencies and test labels.

## Checks appropriate to the change

Documentation/profile-only changes need syntax, schema where applicable, and
reference checks. A history-only reconstruction can be checked with complete
Git tree equality; it does not require interrupting inference to rerun GPU tests.

For source changes, build the host targets before testing them:

```bash
cmake --preset ci
cmake --build build-ci -j 4 --target unit_tests http_server_test serve_test fabric_serve_test scheduler_test roster_check dgpp_serve_app
ctest --test-dir build-ci -L host -LE checkpoint --output-on-failure
```

These tests do not execute GPU work, but configuring/building still requires the
CUDA compiler and native libraries. Do not assume a generic hosted runner can
build them without provisioning those dependencies.

Reserve idle Spark hardware for relevant GPU tests and real-model serving checks.
Rebuild selected targets first and run GPU tests serially. Focused Qwen changes
should cover QSA prefill, FP8 head arithmetic, eager/graph equivalence, four slots,
MTP and prefix reuse as applicable. The world-of-one graph fixtures in
`tests/cuda/qwen_engine_test.cpp` support this machine without a RoCE fabric.
Memory-indexing changes should also run the relevant compute-sanitizer checks.

Report unavailable multi-node/RoCE groups and skipped checkpoint cases explicitly.
A successful single-node test does not validate the multi-node path.

## Live comparison

1. Restart between cold-prefix A/B configurations and ensure no other inference
   traffic is contaminating the measurement. Keep Linux file-cache conditions
   recorded separately from model prefix-cache reuse.
2. Measure C1 decode across prose, code, JSON, math and agent-style text; measure
   cold prefill at 2K/8K/32K and mixed prefill/decode with four-request waves.
3. Check real first-content latency, active-stream gaps, request errors, token
   counter deltas, host MemAvailable, swap and memory-pressure counters. SSE
   comment heartbeats and initial role events are not generated tokens or TTFT.
4. Reuse fixed math/tool/instruction samples with identical output budgets.
   Record truncation separately; longer answers are a workload change, not a
   kernel accuracy improvement. Synthetic tool functions must remain mocks.
5. For attention, context, caching or admission changes, repeat long-context
   retrieval and the three-near-full-context/fourth-short-request capacity test.
   Evaluate prefix reuse over multiple simultaneous multi-turn histories.

Compare distributions/repeated runs rather than selecting the best run. Treat a
new correctness failure, numerical mismatch, request failure or material memory
regression as unresolved until investigated. State workload-dependent tradeoffs
when a throughput gain makes streaming responsiveness worse.

The initial campaign's fixed inputs, stdlib client drivers, raw outputs and
native logs are retained in the separate Spark operations workspace under
`experiments/optimization-2026-09-26/` and `scripts/`. They are not copied into
this source fork. For repeatable A/Bs, use those saved fixtures rather than
regenerating prompts. Public summaries do not substitute for the raw evidence.

## Release gate

Record a dated result, tag the exact tested source and package it using the
upstream release script. Install the bundle and verify that the ordinary launch
command selects it through the site's release pin. Check the running executable,
version, LAN health/model discovery, and bounded streaming/non-streaming replies.
Keep the previous release and its configuration for rollback, and stop any
temporary monitor or benchmark processes afterward.

Publishing a branch is not a deployment. A new source SHA is not automatically
the serving SHA; check the installed release before interpreting live behavior.
