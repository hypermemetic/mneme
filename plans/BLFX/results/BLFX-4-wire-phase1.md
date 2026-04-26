---
title: "BLFX-4 wiring — Phase 1 result (loop scaffold lands; action execution still stubbed)"
date: 2026-04-26
status: partial-success
---

## What was wired

1. `StepDriver` trait refactored to receive a `StepContext` (initial question,
   step index, max steps, history reference) so each driver can choose its
   own prompt strategy. Mock `QueueStepDriver` renders full-history prompts
   for inspection; real `ClaudecodeStepDriver` only emits the *new* content
   per step (claudecode session retains history server-side).
2. `ClaudecodeStepDriver` (substrate-side, `mneme-substrate/src/mneme/runtime/claudecode_step_driver.rs`)
   wraps `Arc<ClaudeCode<P>>` + a session id, sends one `chat()` per step,
   captures `ChatUsage` from the terminal `Complete` event, and accumulates
   it into a `TrialUsage` total over the trial.
3. `claudecode_swarm_runtime::run_iterative_trial` forks the parent session,
   builds a `ClaudecodeStepDriver`, and drives `iterative_trial` from the
   forecast activation. Branches on `TrialParams.iterative_max_steps`:
   `None` → single-shot path (existing behavior); `Some(t_max)` → iterative.
4. `forecast.update` gained `iterative_max_steps: Option<u8>` parameter
   (synapse callable). `Some(0)` is treated as `None` so the wire form
   doesn't need null-vs-zero ceremony.
5. Trial-level error messages now flow into the `swarm_trial` trace entry's
   `args_summary.failure_messages` (and a `tracing::error!`) so failed
   trials don't silently disappear into "AllTrialsFailed".

## Live verification

Bench `bc6f633d-0092-4801-9837-dcaa27cb6103` (T_max=3, K=1, WebSearch only):

```json
{
  "op": "swarm_trial",
  "args_summary": {
    "iterative_max_steps": 3,
    "trials_with_usage": 1,
    "usage_total": {"num_turns": 2, ...},
    "failure_messages": []
  },
  "outcome": "ok",
  "duration_ms": 58147
}
```

Artifact contains a full structured belief (`probability=0.32`, 3
evidence_for, 4 evidence_against, 4 open_questions). The loop drove
multi-turn conversation against the real claudecode session and returned
a parseable belief. **The format contract held end-to-end** — fixing
the original "bare-string action" parse failure required adding a
concrete JSON shape example to every per-step prompt.

## What's still stubbed (the substantive gap)

`agent_loop::execute_action` returns `Observation::Error("not yet
implemented")` for `WebSearch`, `LookupUrl`, `SummarizeResults`,
`FetchTimeSeries`, `FetchWikipediaSection`. This means: when the model
emits `{"action": {"type": "web_search", ...}, "belief": ...}`, the
substrate logs the action but executes nothing. The next step's prompt
includes "Action failed: stub" as the observation. Models then mostly
submit early using training-only knowledge — exactly what we saw above
(2 turns, training-only evidence).

So the iterative loop **scaffold works** but provides **no substantive
information advantage** over single-shot until `execute_action` is wired
to real tool backends. Without that, BLFX-4's claimed -3.8 BI improvement
(per Murphy 2026 ablation) cannot be realized.

This is a separate, well-scoped follow-up ticket — see BLFX-16.

## Recommended next moves

- **BLFX-16** (filed as part of this session): wire `execute_action` to
  claudecode-driven action execution. WebSearch is the priority case
  (covers ForecastBench question types). Source-specific tools from
  BLFX-15 plug into the same surface.
- **Then re-run** this bench (BLFX-4-WIRE-004) and compare turn count,
  evidence sources, and probability vs the single-shot baseline
  (`8feb9552`, `e1e52a17`). Target: turn count > 3; evidence shows real
  URLs in `source` field.
- **Until BLFX-16 lands**, default `iterative_max_steps` to `None`
  (single-shot) in `forecast.update`. Iterative mode is opt-in only.

## Build / test status

- `cargo build --bin mneme-substrate` — green
- `cargo test --lib` — 328 passing (3 new from claudecode_step_driver,
  + 1 new in forecast::activation for resolve)
- `forecast.resolve` (MNEME-23) wired in same commit; calibration store
  grows by one row per call; tests cover artifact-missing and successful
  recording paths.
