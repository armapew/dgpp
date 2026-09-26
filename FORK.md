# Spark maintenance fork

This fork maintains measured improvements for a headless, single-node NVIDIA
GB10 serving `nvidia/Qwen3.8-Flash-Next-NVFP4`. The main consumer is an agent on
another machine. Prioritize correctness and checkpoint fidelity, then reliable
streaming, prompt reuse and throughput. Size memory headroom against measured
inference peaks on the dedicated headless host. Keep general fixes suitable for
eventual upstream submission.

Upstream is [HawkBearPig/dgpp](https://github.com/HawkBearPig/dgpp). The maintained
fork is [armapew/dgpp](https://github.com/armapew/dgpp). Repository publication and production deployment are
separate operations.

## Branches and releases

| Ref | Purpose |
| --- | --- |
| `master` | Unmodified upstream mirror; fast-forward from `upstream/master` |
| `spark` | Maintained source, tests, deployment example and fork documentation |
| `work/*` | Improvements and upstream integration candidates |
| `archive/optimization-20260926` | Original experiment history, including reverted candidates |
| `spark-2026.09.26.1` | Immutable tag for the original validated source `d44bebb61205` |
| `spark-2026.09.26.2` | Validated upstream integration and profile catalogue fix at `f5739da0db70` |
| `spark-2026.09.26.3` | Checkpoint dense weights, optional separate request execution and upstream through `652cd40` |

`origin` points to the fork and `upstream` to the original repository. Use `spark`
as the fork's default branch. Keep released tags fixed and merge shared branches
without rewriting their history. Experimental branches can be discarded after
their results and retained changes have been recorded.

The initial `spark` source was reconstructed into focused commits. At
`3c9849451615ba771a4936a231b71cd1ceb5e2dd`, its complete Git tree is identical to
the tested `d44bebb612051a9581803479e6ec456c2e5c62ae`:
`46bce6549067d59f60c99c2aa025416b05704ffa`. The initial fork setup then added
documentation and a deployment example. Rebuilding a new commit changes the
reported version; this does not retag or replace the already validated binary.

Release `spark-2026.09.26.2` incorporates upstream PR #54 and serves as
`0.1.0+gf5739da0db70`. It keeps the initial engine settings and checkpoint history
defaults. See [the integration record](benchmarks/results/2026-09-26-spark-upstream-history.md)
for native checks, live request comparisons, the template-name correction and
the new history-control checks.

The next accuracy profile and its limitations are recorded in
[the accuracy result](benchmarks/results/2026-09-26-spark-accuracy.md). It restores
checkpoint dense weights and uses separate request execution. Fixed-history
concurrency checks improved, but the complete TC-63 scenario still reproduced a
reasoning loop. Do not describe the new profile as a general loop cure.

## Retained differences

| Change | Reason and validation |
| --- | --- |
| Context-bounded QSA scoring and visible prefill stripes | Avoid sizing/launching per-request attention work for the whole shared pool |
| 4096-token internal Qwen prefill cap | Now supplied by upstream PR #58; full-length prefill checks retained |
| Compact FP8 serving logits | Keep vocabulary-head storage proportional to selected serving rows while preserving numerical dispatch |
| Tiled QSA temporary workspace | Bound score/reduction scratch; full/256/8-row numerical comparisons retained |
| Configurable SSE comment heartbeats | Upstream PR #61 replaces the local implementation; default 30 seconds, socket behavior and cancellation tested |
| Optional separate request execution | `graph_batch_min_live = max_concurrency + 1` disables row batching and grouped prefills while preserving all request slots |
| World-of-one graph and kernel checks | Exercise four slots, MTP depths, unequal output lengths, compact heads and routing edge cases |
| Resolved configuration fixture correction | Align a stale fixture with the existing example output limit |

The production W4A4 MoE kernels match the upstream base. Kernel experiments that
did not show dependable live improvements are present only in the archive
history. The model checkpoint has not been changed.

The 4096-cap experiment was informed by [upstream PR #58](https://github.com/HawkBearPig/dgpp/pull/58).
Its other example settings were assessed independently; our profile retains full
admission and the NVIDIA checkpoint's existing expert format.

## Deployment profile

Use [the Spark profile](deploy/cluster_qwen-3.8-flash-next_nvfp4_w1_spark.example.json)
as a template. It records four slots, 850048 BF16 KV tokens, MTP depth 1,
4096/4096 idle/busy prefill tokens, 3 GiB prefix snapshots and full admission.
Dense projections and the head retain their checkpoint BF16 weights. The
checkpoint's NVFP4 experts and native FP8 components remain as shipped.
`graph_batch_min_live: 5` selects separate prefill forwards and scalar decode
graphs for the four request slots. This sacrifices aggregate throughput to keep
decode arithmetic independent of other live slots. It does not guarantee that
every prompt/cache history produces identical output or that greedy reasoning
can never loop. Consult the recorded accuracy validation before interpreting it.
The per-request model context ceiling is 262144 tokens including output.

The example intentionally has no release pin: add the exact installed release
to the site's deployment JSON. Keep credentials, addresses, checkpoint paths,
SSH details and `.env` in the separate site operations workspace. Copy the
example to a local deployment file before adapting it.

Three full-size context reservations fit the shared pool; four do not. A fourth
short request can run alongside three populated contexts when its output
reservation fits. Larger busy chunks can delay existing streamed output.

## Incorporating upstream changes

Start with a clean working tree. Fetch upstream and advance the mirror, then
integrate into a new candidate branch:

```bash
git fetch upstream
git switch master
git merge --ff-only upstream/master
git switch spark
git switch -c work/upstream-YYYYMMDD
git merge upstream/master
```

Choose a fresh branch name for each update. Inspect overlapping fixes and remove
local patches when upstream supplies equivalent behavior. A conflict-free merge
still needs the validation described below. Put the upstream revision, retained
differences, test evidence and performance comparison in the candidate PR.

After validation, merge into `spark`, tag the exact tested commit, package the
release, and update the site's release pin. Retain the previous binary and
configuration together for rollback. Deploy during an agreed maintenance window.

Do not run `scripts/ci-local.sh` against a serving Spark. GPU/RDMA tests require
idle hardware, as explained in [CONTRIBUTING.md](CONTRIBUTING.md). Routine source
sync and documentation work can happen on the workstation.

## Validation and evidence

Read [the validation checklist](docs/fork-validation.md) before changing source
or promoting an upstream update. The initial measurements, limitations and
source identities are in [the baseline record](docs/fork-baseline-2026-09-26.md).

A useful upstream contribution contains a focused patch, a reproducer or test,
the relevant hardware results and its tradeoffs. Keep fork-specific deployment
preferences separate from general engine fixes.
