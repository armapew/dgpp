# Checkpoint dense weights and separate request execution on GB10

The Spark profile now prioritizes checkpoint fidelity and repeatable request
execution. Optional FP8 dense conversion is disabled, and the opt-in scalar mode
prevents live request count from selecting a different decode shape or combining
short prompt forwards. This is slower and is **not a general fix for reasoning
loops or a demonstrated increase in the full benchmark score**.

Upstream is integrated through `652cd40`: Unicode/UTF-8 correctness, configurable
SSE keepalives, lazy role preambles, the prefill changes and Prometheus MTP
counters. The local duplicate heartbeat implementation was removed. The
upstream keepalive default is 30 seconds.

## Selected profile

Same NVIDIA Qwen3.8 Flash Next NVFP4 checkpoint; native expert formats remain
unchanged. Dense projections and the output head use checkpoint BF16 weights.

| Setting | Value |
| --- | --- |
| Request slots | 4 |
| Shared BF16 KV pool | 850048 tokens |
| Context per request, input plus output | 262144 tokens |
| MTP | Enabled, depth 1 |
| `dense_weights` / `bf16_weights` | `checkpoint` / `checkpoint` |
| Idle/busy prefill budgets | 4096 / 4096 |
| `graph_batch_min_live` | 5: separate prefills and scalar decode at every occupancy |
| Prefix snapshot budget | 3 GiB |
| Admission | Full reservation |

The ordinary graph batching default is unchanged. The new mode is optional and
selected only by the profile. It retains concurrent requests but gives up the
throughput benefit of row batching. Fixed chunk budgets are a separate setting;
cache histories can still affect floating-point results.

## Consistency and quality checks

Model checks used `c45454f`; the later upstream merge only adds an HTTP metrics
route. All diagnostic quality requests used temperature 0, seed 42, xhigh and
the original 65536-token output reservation.

- BF16 alone retained different C1/C3 output. Fixing the prefill chunk size
  alone also retained the difference. Separate execution matched isolated and
  concurrent output on the same fixed history.
- Six complete replays of the previously failing TC-63 turn produced identical
  993-token responses: cold, warm, three concurrent cached copies, and a target
  surviving neighboring request retirement and slot reuse. Cached runs reused
  3808 of 3815 prompt tokens. A separate bounded comparison matched all five
  512-fragment outputs and its slot-reuse target.
- Four synthetic coding task types passed executable checks: environment
  parsing, cursor pagination, versioned updates and linked digit construction.
  One case needed a clean rerun after the test runner's missing standard Python
  `ord` builtin was corrected; its original implementation also passed with
  the corrected runner. Retained failed traces are not engine failures.
- Escaped and raw non-BMP Unicode prompts produced identical correct API text.
- A targeted actual benchmark passed TC-66 and TC-88, but **TC-63 later entered
  another reasoning loop**. It was a new request admitted after the other
  scenarios completed; a 47-token cycle covered 4569 of 6229 captured tokens.
  The diagnostic was cancelled. There is no completed three-case score and no
  new full 92-case score. This failure does not require active row batching or
  extra FP8 dense conversion; cache/state effects and model behavior are not
  independently ruled out.

One early comparison included unrelated traffic and was discarded. The clean
repeats checked request-counter deltas. Native checks covered 27 host CTest
groups, single-node MTP/eager agreement, four slots, and explicit scalar-mode
retirement/reuse with MTP on/off and checkpoint/FP8 dense weights. The final
release gate rebuilds the merged tree and reruns host and scalar GPU checks.
Multi-node/RoCE behavior is not validated by this single-node experiment.

## Capacity and performance

Three distinct 261862-token prompts returned the correct retrieval codes.
Reusing those contexts together with a fourth short generation reached four
active requests without prefill; all four answers were correct. The long
requests reused 99.996% of their prompts. This preserves three near-256K
contexts, not four full contexts.

During the approximately ten-minute capacity check, available RAM reached a
minimum of 2.42 GiB, with no OOM or engine failure. Swap traffic was about
134 MiB out and 14 MiB in; peak swap use was about 201 MiB. The unchanged
startup plan includes a 4 GiB allowance, but actual post-load headroom is
smaller. These results apply to the measured dedicated headless host.

Matched bounded throughput cases, temperature zero and no thinking, with MTP
depth 1 and the same pool/cache budgets in both profiles:

| Workload | Previous FP8/batched profile | BF16/separate requests |
| --- | ---: | ---: |
| Code decode | 46.0 tok/s | 30.8 tok/s |
| JSON decode | 49.8 tok/s | 33.4 tok/s |
| Agent prose decode | 44.8 tok/s | 29.3 tok/s |
| About 2K cold prefill | 1540 tok/s | 1459 tok/s |
| About 8K cold prefill | 1722 tok/s | 1295 tok/s |
| About 32K cold prefill | 1814 tok/s | 1386 tok/s |
| Four 2K prompts, 128 output tokens each | 13.29 s total | 25.91 s total |

These are single measurements across fixed workloads, not a statistical full
benchmark. Raw requests, responses, counters, invalid comparisons, memory
samples and rollback artifacts are retained in the separate Spark operations
workspace under `experiments/accuracy-upgrade-2026-09-26/`.
