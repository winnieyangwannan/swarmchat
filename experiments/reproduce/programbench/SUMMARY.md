# ProgramBench Hex Reproduction Summary

Last updated: 2026-11-06 03:05 PST / PT

## Scope

This reproduction tested ProgramBench with swarmchat on one task:
`sitkevij__hex.61ae69b`.

Configuration:

- Model: `gpt-6-luna`
- Provider path: Codex app-server `relay` through the ProgramBench host relay to `https://api.meta.ai`
- Agent kind: `codex`
- Team shape: `flat8`, eight agents in one swarmchat channel
- Reasoning effort: `xhigh`
- Canary budget: 0.25 hours
- Full budget: 6 hours
- Snapshot cadence: 30 minutes
- Task parallelism: 1

## Staged Result

- Smoke and dry-run gates passed after local prerequisite fixes.
- Diagnostic canary11 passed real orchestration: successful model relay traffic, code-mode command execution, shared repository commits, and packaged submission.
- Canary11 scoring completed with the fixed evaluator environment:
  - ProgramBench summary score: `99`
  - Raw results: 864 passed, 13 failed, 0 branch errors, 0 warnings
  - Evidence: `/home/winnieyangwn/swarmchat/experiments/reproduce/programbench/evidence/stage4-canary11-eval-with-proxy-no-xet-podman-wheelhouse-repro-flat8-hex-gpt6-luna-20261105-205816-PST.log`

## Full Run Result

The full run completed at the time limit and produced a final submission.

- Runtime status: `TimeLimit`
- Runtime seconds: `21696`
- Final submission: `/home/winnieyangwn/swarmchat/extras/programbench/runs/repro-flat8-hex-gpt6-luna-full/submissions/sitkevij__hex.61ae69b/submission.tar.gz`
- Final submission size: 1.3M
- Snapshots: `00030`, `00060`, `00090`, `00120`, `00150`, `00180`, `00210`, `00240`, `00270`, `00300`, `00330`
- Shared repo commits at final inspection: 126
- Model relay request log entries: 12,551 total, 5 transient HTTP failures
- Codex code-mode warnings: 0
- Codex turn errors: 0
- Stage 5 evidence: `/home/winnieyangwn/swarmchat/experiments/reproduce/programbench/evidence/stage5-full-run-artifact-inspection-20261106-030305-PST.log`

Final scoring passed:

- ProgramBench summary score: solved (`✅`), average `100`
- Filtered test count shown by ProgramBench: 823 tests
- Raw eval JSON: 877 passed, 0 failed, 0 branch errors, 0 warnings
- Executable hash: `04f4f2c7b8fef21505eade9b3949e1a8dbedfa859155a7e987287a15c1c3d695`
- Stage 6 evidence: `/home/winnieyangwn/swarmchat/experiments/reproduce/programbench/evidence/stage6-full-eval-repro-flat8-hex-gpt6-luna-20261106-030405-PST.log`
- Eval JSON: `/home/winnieyangwn/swarmchat/extras/programbench/runs/repro-flat8-hex-gpt6-luna-full/submissions/sitkevij__hex.61ae69b/sitkevij__hex.61ae69b.eval.json`

## Operational Notes

The successful setup depends on the local experiment wrappers and patches recorded
in `PLAN.md` and `LESSON.md`, notably:

- Podman compatibility wrappers for ProgramBench run/eval paths.
- Direct DotSlash Codex release launch inside task containers.
- Mounted `codex-code-mode-host`.
- Experiment-local `/etc/codex` configured for `relay`.
- Model API relay adapter for proxy support, analytics 204 handling, and flattened Codex tools.
- Eval environment using `with-proxy`, `HF_HUB_DISABLE_XET=1`, rootless Podman tar ownership handling, and the offline pytest wheelhouse.

Reduced-budget canary metrics are diagnostic only. The final result above is from
the full six-hour hex-only configuration requested for this run.
