# Spark maintenance fork

This fork maintains measured improvements for a headless, single-node NVIDIA
GB10 serving `nvidia/Qwen3.8-Flash-Next-NVFP4`. The main consumer is an agent on
another machine. Favor reliable streaming, prompt reuse, decode throughput and
memory headroom. Keep general fixes suitable for eventual upstream submission.

Upstream is [HawkBearPig/dgpp](https://github.com/HawkBearPig/dgpp). The intended
fork is `armapew/dgpp`. Repository publication and production deployment are
separate operations.

## Branches and releases

| Ref | Purpose |
| --- | --- |
| `master` | Unmodified upstream mirror; fast-forward from `upstream/master` |
| `spark` | Maintained source, tests, deployment example and fork documentation |
| `work/*` | Improvements and upstream integration candidates |
| `archive/optimization-20260926` | Original experiment history, including reverted candidates |
| `spark-2026.09.26.1` | Immutable tag for the original validated source `d44bebb61205` |

`origin` points to the fork and `upstream` to the original repository. Use `spark`
as the fork's default branch. Keep released tags fixed and merge shared branches
without rewriting their history. Experimental branches can be discarded after
their results and retained changes have been recorded.

The initial `spark` source was reconstructed into focused commits. At
`3c9849451615ba771a4936a231b71cd1ceb5e2dd`, its complete Git tree is identical to
the tested `d44bebb612051a9581803479e6ec456c2e5c62ae`:
`46bce6549067d59f60c99c2aa025416b05704ffa`. Subsequent fork setup adds only
documentation and a deployment example. Rebuilding a new commit changes the
reported version; this does not retag or replace the already validated binary.

## Retained differences

| Change | Reason and validation |
| --- | --- |
| Context-bounded QSA scoring and visible prefill stripes | Avoid sizing/launching per-request attention work for the whole shared pool |
| 4096-token internal Qwen prefill cap | Enables the selected larger idle chunks; full-length prefill checks retained |
| Compact FP8 serving logits | Keep vocabulary-head storage proportional to selected serving rows while preserving numerical dispatch |
| Tiled QSA temporary workspace | Bound score/reduction scratch; full/256/8-row numerical comparisons retained |
| SSE comment heartbeats every 15 seconds | Keep silent prefill/queue streams active; socket behavior and cancellation tested |
| World-of-one graph and kernel checks | Exercise four slots, MTP depths, unequal output lengths, compact heads and routing edge cases |
| Resolved configuration fixture correction | Align a stale fixture with the existing example output limit |

The production W4A4 MoE kernels match the upstream base. Kernel experiments that
did not show dependable live improvements are present only in the archive
history. The model checkpoint has not been changed.

The 4096-cap experiment was informed by [upstream PR #58](https://github.com/HawkBearPig/dgpp/pull/58).
Its other example settings were assessed independently; our profile retains full
admission and the NVIDIA checkpoint's existing expert format.

## Deployment profile

Use [the Spark profile](deploy/cluster_spark_qwen-3.8-flash-next_nvfp4_w1.example.json)
as a template. It records four slots, 850048 BF16 KV tokens, MTP depth 2,
4096/1024 idle/busy prefill tokens, 3 GiB prefix snapshots and full admission.
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
