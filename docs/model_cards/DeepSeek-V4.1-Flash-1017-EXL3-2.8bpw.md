# DeepSeek-V4.1-Flash-1017 EXL3 2.8 bpw two-Spark text baseline

This baseline targets
[`jayleaton/DeepSeek-V4.1-Flash-1017-EXL3-2.8bpw`](https://huggingface.co/jayleaton/DeepSeek-V4.1-Flash-1017-EXL3-2.8bpw)
at revision `859396591c340195526aaf61f94b4ad5323a5e18` on two GB10 nodes.
The operator accepted this text baseline on 2026-11-01 after the full
evaluation recorded below. DeepSeek vision, Responses/client integration and
further performance changes remain separate future work.

## Weight contract

The checkpoint has no `exl3_k_map.json`. The config reader discovers widths
from safetensors headers; the binding table still supplies and checks every
architectural shape. Gate/up widths agree within each expert, while down
projections can be wider, and experts within a layer can differ. The seven
backbone gate/up/down patterns are 2/2/2, 2/2/3, 3/3/3, 3/3/4, 4/4/4,
4/4/5 and 5/5/5. Shared experts use 5 or 6 bits. Dense projections carry
their own widths, including 8-bit indexer keys and the 6-bit output head.

The native BF16 routers/heads and FP32 biases retain their stored precision.
The other EXL3 pack's F16/F32 native representations remain accepted only
through the existing exact-conversion checks. Eleven extra BF16 copies of
compressor/indexer matrices are validated and skipped when their EXL3 group
is present, matching the pack's TensorFold selection. No requantization or
expert pruning is part of this port.

The CUDA core includes 2/5-bit forms. Heterogeneous MoE launches read each
selected expert's width from its view; all threads in a CTA take the same
branch into the existing per-width arithmetic. Uniform packs retain their
specialized width-pair paths. Prefill, decode, graph tables and TP slices
must all pass the mixed-width fixtures before serving this candidate.

## Serving profile

The [two-node template](../../deploy/cluster_deepseek-v4.1-flash-1017_exl3-2.8bpw_w2.example.json)
uses four slots, a 1,048,576-token shared pool, 2,048-row prefill budgets,
a 1 GiB prefix arena and confidence-scheduled DSpark verification at depths
1, 3 and 5. Its cost model uses a 28.5 ms base plus 4.7 ms per verify row,
seeded from two-node step measurements; the multiplier adapts and sampled
requests use the existing 0.93 acceptance scale. These are deployment-specific
prices, not the four-node 3.5 bpw template's costs. The prefix creation floor
is 256 tokens; cache capacity and 128-token snapshot alignment are unchanged.
The family allocates a 4,096-row forward workspace even with these 2,048-token
per-tick caps. Snapshot/pool allocations remain conservative. The 1M pool is
shared across requests; its API request ceiling does
not mean that four independent 1M contexts fit.

DeepSeek's existing DGPP cache formats remain FP4-block main compressed KV,
FP8-block sliding windows and indexer FP4 quantization stored as FP8 codes
plus row scales. These differ from TensorFold's
default FP8 split main cache; the same weight pack does not imply identical
cache arithmetic or output. Keep that distinction in evaluation records.

The original Engram tables are external, selected by `engine.engram_source`
at revision `1bae0e1088993b9b82de08f59de5be5a34e42170`. Only the table tensors
from shards 47/48 are admitted. Generate their hash sidecar with
`tools/dsv41_engram_tables.py` against the served tokenizer. The sidecar's
token map and multipliers must match the reference.

Bind the private API to loopback and retain the site's authenticated proxy.
Temperature-zero comparisons use seed 42 and `reasoning_effort: "xhigh"`.
Include sampled traffic, cached follow-ups and cancellation in focused checks.
Full quality benchmarks belong to the operator after the candidate is ready.
The pack has no `generation_config.json`. To retain the selected ordinary
sampling defaults, use the normal launcher with
`--knobs '--temperature 1 --top-p 0.95 --top-k 20'`; each request can override
them. Without those flags DGPP falls back to greedy defaults.

## Validation scope

Build `038c897ad590`, based on upstream `92303ef9`, passed the dense EXL3
oracle and row-invariance tests,
the mixed MoE oracle/prefill/decode/graph-table tests, and the EXL3 loader
tests. The two-rank synthetic model comparisons pass the existing numerical
budgets at 24 and 70 rows. A supplementary four-rank run of the mixed fixture
fails the first-layer routing-agreement gate and, at 24 rows, the aggregate
layer-local routing-flip limit. Its per-layer numerical errors remain within
the tolerance. Those budgets were not loosened; the dedicated mixed-pack TP
test covers the requested two-node deployment. Four-node support for this
mixed pack is not validated by this candidate.

The actual checkpoint binding matched all 192,510 tensors, including the four
external Engram tensors, with no missing tensors or dtype/shape mismatches.
The token map and hash multipliers match the reference's prepared Engram
asset. Physical two-node eager/graph collectives passed; bulk payload checks
exercised both configured RoCE lanes. The existing uniform EXL3 two-/four-rank
regression checks also passed.

Focused serving checks on two GB10 nodes passed arithmetic, JSON, Python code,
tool-call/result replay, retrieval at 19,854 and 99,854 prompt tokens, cached
follow-ups, cold C1/C2/C4, cached C4, mixed temperature-zero/0.6 traffic and
stream cancellation followed by successful slot reuse. The four-slot decode
graph was exercised. All non-cancelled requests completed without truncation;
the service reported no engine/request failures and both serving processes
stayed at zero swap. The two ranks' operation streams matched byte-for-byte
at clean shutdown. Restarting from all 43 resident-image layers reproduced the
same greedy message, and strict JSON-schema output passed after restart.

These initial checks are bounded correctness and deployment checks, not a full
quality or performance campaign. Quality above roughly 100K context,
maximum-pool C4, vision and Responses/client integration remain unvalidated.
The subsequent operator-run text evaluations are recorded below.

## Serving-contract corrections

Build `9eb7718e21fd` adds native response-format schema prompting after the
tool block and enforces required/named DSML tool openers after reasoning ends.
The renderer's cache-identity revision is bumped. Model weights, routing,
KV formats, sampling and the initial profile above are unchanged.

Validation passed 122 HTTP/service tests, 26 grammar tests, the original 20
reference prompt fixtures and four schema fixtures (exact bytes and tokenizer
IDs), plus required/named calls with the actual tokenizer. On two GB10 nodes,
the unmodified tool-eval-bench 3.0.1.dev7+g887b53757 capability probe completed
with a valid call and arguments in its first 256-token attempt. At T0, seed 42,
xhigh and C3, TC-45/64/65/66/69 passed; TC-67/68 remained partial for unnecessary
tool calls despite valid final JSON. No full-suite score is inferred.

Focused C4 checks also covered sampled/streamed required and named tools,
automatic replies, schema output with tool-result replay, cached follow-ups
and cancellation/slot reuse. Both ranks stayed at zero process swap, with no
service failures. Vision and client integration remain later work; focused
checks alone do not establish full-suite quality.

The operator's subsequent full run on this build scored 199/208 (97 passes,
five partials, two failures), with all 104 scenarios scored using the same
evaluator, T0, seed 42, xhigh and C3. This matches the comparison TensorFold
run's aggregate score, with different per-case outcomes. Performance parity
and general quality parity are not established.

A remaining JSON/tool interaction was isolated with an identical native prompt:
the constrained Chat Completions path emits a calculator call, while the
unconstrained continuation has identical reasoning followed by fenced JSON.
The native render matches the reference byte-for-byte. This motivated the raw
JSON clarification below; structured-output validation and the evaluator remain
unchanged.
The subsequent serving/profile changes are recorded below.

## Raw JSON and adaptive verification

Build `3b3f4bc497f3` makes the raw-JSON contract explicit in system content
whenever a response schema is requested. This resolves the reproduced cold
fence/tool check without weakening the grammar, adding output rewriting or
changing ordinary chat prompts; the later full run still has a TC-67 partial.
The renderer revision is bumped; 20 original and four
schema prompt fixtures match the reference encoder with that API instruction.

The operator's evaluator subset, unchanged at T0/seed 42/xhigh/C3, passes
TC-45/64/65/66/67/69; TC-68 remains partial for an unnecessary search. The
required-tool capability probe succeeds in 40 tokens with valid arguments.
The full benchmark below records the broader outcome on this build.

The selected confidence schedule reproduced four baseline capped greedy
transcripts exactly in C1/C3 checks. C2 and C4 mixed-temperature checks,
sampled named tools and JSON streaming, cancellation/slot reuse, 20K/100K
retrieval and cache reuse also passed. A lower prefix floor improves repeated
short requests but adds snapshot work to their first cold request. The one-GiB
arena was retained. Both serving processes remained at zero swap in the
focused validation. These checks do not establish universal numerical or
quality parity across batching shapes, histories or sampling seeds.

A rank-0 prefill profile remains dominated by EXL3, attention and collective
work. An idle 4K chunk trial showed only a small one-pass gain; the selected
profile retains 2K busy/idle caps. No kernel arithmetic, quantization, expert
routing, KV precision, model weights or sampling defaults changed.

## Accepted text baseline

The operator accepted this version as the stable text baseline on 2026-11-01.
The preserved branch is
`baseline/deepseek-v4.1-flash-1017-2.8bpw-tp2-20261101` in `armapew/dgpp`.
It contains the complete implementation, portable deployment template and
validation notes. Future work should branch from this baseline; retain the
existing `spark` lineage separately.

| Reproduction identity | Value |
| --- | --- |
| Upstream base | `92303ef9ddcdd67ae9acb4e8e199f5c6219b3d14` |
| Exact engine commit | `3b3f4bc497f3cc1540bbe9909412a4d74bc8037f` |
| Immutable engine tag | `spark-deepseek-q28-text-2026.11.01` |
| Installed release | `0.1.0+g3b3f4bc497f3` |
| Binary SHA256, both ranks | `0f247418037cc1b1a9fa92a7ff424bf83fcece4dd5e32ff2a8bb63fc21a6db83` |
| Release package SHA256 | `6338a6df4538ba6a742f584ad2cc041399e157e3d06e4ebca95059071ee406ee` |

The branch includes later documentation commits. Rebuild the engine tag above
to reproduce the tested implementation. Site credentials, local deployment
addresses, model weights and binary packages are not part of this public
source branch. The operator retains the exact package, source bundle,
deployment JSON, private environment, launch defaults and checksum manifest
in the deployment's recovery set. Restore the matching JSON and environment
together with the release; preserve the authenticated proxy and loopback-only
backend.

The operator's full quality run
`2026-10-31T23-46-01.768852Z_d262f59c` scored **201/208 (97/100): 98 passes,
five partials and one failure**, with all 104 scenarios scored and no
infrastructure exclusions. The evaluator was unmodified
`tool-eval-bench 3.0.1.dev7+g887b53757`, at temperature zero, seed 42, xhigh,
parallelism three and `max_tokens=65536`. The engine retained C4 admission,
the 1M shared pool, 2K prefill caps and adaptive depths 1/3/5 described above.

The two-point gain over the preceding 199/208 run came from TC-104 passing.
TC-57/60/63/67/68 remained partial, and TC-92 still failed tenant isolation.
TC-67's focused pass did not generalize to the full concurrent run; it again
called an unnecessary calculator despite schema-valid final JSON. Preserve
that case in future concurrency checks. This acceptance freezes a known
working baseline with these limitations; it does not establish universal
quality parity or resolve every reasoning failure.

The exact release also passed the focused retrieval, sampled concurrency,
streaming, cancellation and restart checks above. Available memory during
those checks stayed above 8.84 GiB on the head and 8.74 GiB on the peer, with
both serving processes at zero swap. These measurements cover bounded
20K/100K and C4 traffic, not four maximum-context requests or future vision.
No new throughput campaign is claimed by this preservation step. Performance
parity with the retained TensorFold deployment remains unestablished.
