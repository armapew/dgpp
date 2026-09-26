# Spark fork baseline — 2026-09-26

The initial fork captures a validated single-GB10 configuration serving
`nvidia/Qwen3.8-Flash-Next-NVFP4`. The upstream base is
`063dc96fc76991df40ecdbd4c5f99bcc603a0a8d`. The tested source is
`d44bebb612051a9581803479e6ec456c2e5c62ae`, preserved under tag
`spark-2026.09.26.1` and archive branch `archive/optimization-20260926`.

The clean source commit `3c9849451615ba771a4936a231b71cd1ceb5e2dd` has the same
complete tree as that tested source, including all tests. Its later maintenance
documentation/profile additions do not change engine code. No rebuild or GPU
benchmark was performed solely for this history cleanup; the installed release
remains `0.1.0+gd44bebb61205`.

## Final matched performance

Both variants ran the same 29-record, 37-request suite after a fresh start.
The original already had four slots and an 850048-token pool. The final profile
selects MTP 2, 4096/1024 idle/busy prefill and 3 GiB prefix snapshots.
Decode/prefill figures below are medians of three repetitions. Prefill uses
server counters with no reusable model KV prefix; Linux file cache was not flushed.

| Measurement | Original | Final |
| --- | ---: | ---: |
| Prose decode, tok/s | 43.33 | 44.45 |
| Code decode, tok/s | 46.18 | 54.23 |
| JSON decode, tok/s | 49.80 | 60.30 |
| Math decode, tok/s | 46.95 | 53.08 |
| Agent-style decode, tok/s | 44.54 | 49.01 |
| 2K prefill, tok/s | 1399 | 1450 |
| 8K prefill, tok/s | 1468 | 1581 |
| 32K prefill, tok/s | 1307 | 1585 |
| 32K first content, seconds | 25.205 | 20.759 |
| Four 8K-input / 256-output requests, seconds | 58.709 | 37.191 |
| Startup memory plan, GiB | 106.44 | 104.78 |

The matched mixed workload completed in 21.24 versus 28.15 seconds, while the
active stream's maximum content gap increased from about 430 to 808 ms. Short
final performance windows recorded about 4.50 MiB swap out for the candidate
and 142.84 MiB for the original; those system counters do not identify the process
causing pressure. The near-capacity runs were effectively swap-free.

## Correctness and capacity

- Fixed 64-question GSM8K sample, reasoning disabled and 1024 output tokens:
  59/64 for both; all 64 responses were identical. More output budget completed
  one truncated answer correctly. This was not a change in model accuracy.
- Full 69-scenario mock tool evaluation: 121/138 points for both (88/100 rounded).
- First 32 IFEval prompts: 27/32 prompts, 43/48 constraints for both; same failures.
- Structured extraction: 100/100 correct at concurrency four.
- Long-context retrieval: 18/18 correct, including near-262144-token contexts.
- Three distinct 261862-input contexts were populated, then decoded concurrently
  with a fourth short request. Peak pool use was 788480/850048 tokens; minimum
  available host memory was about 10.34 GiB. Four full contexts do not fit.
- Prefix snapshots: 20/20 follow-up hits with 3 GiB versus 10/20 with 1.5 GiB in
  one synthetic agent workload; a second longer run achieved 28/28.
- Silent-stream heartbeats passed a repeated long-prefill/queue test over the
  same LAN path where earlier idle connections stalled.

These are finite synthetic samples, not a general accuracy guarantee or a
measurement of the actual consumer agent. The four-context test used small
explicit output caps; default output reservations can change admission.

All 27 host groups passed after correcting a stale resolved-config fixture.
Targeted Qwen GPU, numerical equivalence, world-of-one graph and sanitizer checks
passed. Three multi-rank/RoCE groups were unavailable on the single node and are
not counted as passes. The final HTTP-only change passed the relevant host and
API checks.

## Recovery identities

- Original binary SHA-256:
  `23a6ad31defe5ca8bb33fd86a62cbd8318f3c85a2eceaef2c31696159be03c4a`.
- Tested build binary SHA-256:
  `8adafb7437febdd24cb75528e1c956ae983f09886059b012d9e8f623fed36747`.
- Installed release binary SHA-256:
  `9150e1dde71473ed995b6ebe27f157ff67209e41b41e9bad0c5e9244016868f7`.
- Release tarball SHA-256:
  `110fec917b0bf527f24795fcf0240b2695445c507320c9d52fabee35be84b83c`.

Packaging changes the executable's runtime library path for bundled libraries.
The tag, configuration profile and source archive preserve the baseline; the
separate site runbook holds installed paths and rollback commands. No model
weights, credentials or site environment files are part of this fork.
