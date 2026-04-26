---
id: MNEME-24
title: "Parallelize trials in claudecode_swarm_runtime::trial"
status: Complete
type: implementation
blocked_by: []
unlocks: [BLFX-13]
confidence: high
severity: High
---

## Problem

`claudecode_swarm_runtime::trial` runs the K trials sequentially in a `for`
loop. Each trial takes 1.5–3 minutes; K=5 takes 5× wall clock. The paper
runs trials in parallel (that's the entire point of fan-out).

Measured (wall-clock per trial, single-shot mode, Sonnet, WebSearch):

- 1 trial: ~91-157 s
- 2 trials sequential: ~204 s (~102 s/trial — close to 2× one trial)
- 5 trials would be ~10–15 minutes serial; ought to be ~2–3 minutes parallel.

This is the single biggest blocker on benchmark wall-clock.

## Required behavior

Replace the `for trial_index in 0..params.n` loop with a parallel
fan-out using `futures::future::join_all` (or `try_join_all` for
short-circuit-on-first-failure semantics — but we don't want that;
each trial's failure should still produce a `TrialFailure` and the
batch should complete).

Each parallel trial:
- Forks from the same parent session (claudecode supports concurrent
  forks; verify).
- Runs `run_one_trial` or `run_iterative_trial` independently.
- Returns its own `Result<(String, Option<TrialUsage>), String>`.

Order in `successes` / `failures` does not need to match `trial_index`
ascending — but we should preserve `trial_index` on each record so callers
can recover the originating index.

## Acceptance criteria

1. K=5 wall-clock ≤ 1.5× single-trial wall-clock (allowing for some
   coordination overhead).
2. All existing single-trial and stub tests still pass.
3. New test: mock runtime with K=4 returns 4 successes; verify wall-clock
   is dominated by the slowest trial, not the sum.
4. No deadlocks or session-name collisions under K=8 stress test.
5. `cargo test --lib` passes.

## Risks / open questions

- **claudecode session concurrency.** Forking the same parent N times
  in parallel — does the underlying CLI handle this safely? If forks
  serialize internally, we get no speedup. The spike is "verify with
  K=4 timing test." If concurrency is broken, we add a serial-fork
  + parallel-chat pattern as fallback.
- **Resource contention.** N parallel `claude-code` subprocesses ≈ N×
  RAM and CPU. K=8 might OOM a small machine. Probably fine on dev
  hardware; revisit if it bites.

## Out of scope

- Question-level parallelism (separate ticket, MNEME-25).
- Throttling / queue management for very high K (defer until proven needed).

## Completion

Tests pass + wall-clock measurement committed in result doc; status
flips to `Complete`.
