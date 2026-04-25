---
id: MNEME-15
title: "Calibration loop closure (resolve forecasts → update Platt params)"
status: Pending
type: implementation
blocked_by: [MNEME-8]
unlocks: []
confidence: low
---

## Problem

`forecast` (MNEME-6) ships with identity calibration (no shrinkage from learned bias). Once ~10 forecasts have been resolved by the user, the system has enough data to fit Platt-scaling parameters. This ticket closes the loop: a `forecast.resolve` method that records ground truth, plus a calibration job that reads `programs/_calibration/history.jsonl` and updates `programs/_calibration/bias.json`.

## Context

The BLF paper's calibration step requires (predicted, actual) pairs. For binary forecasts the actual is 0 or 1; the predicted is the most recent `probability` for that question before resolution. Platt scaling fits `p_calibrated = sigmoid(a * logit(p_raw) + b)` to minimize log loss.

## Evidence

This ticket is `low` confidence because:

1. The cold-start period (first 10 forecasts) means calibration is identity for early users — does the architecture handle that gracefully without producing falsely-precise probabilities?
2. The calibration parameters need to be domain-stratified (forecasts about software shipping vs. forecasts about external events have different bias) — the MVP punts on this and uses one global Platt fit.
3. Resolution UX (asking the user "did this happen?") is harder than it sounds — many forecasts the user wrote will get forgotten before deadline.

The `low` confidence is *correct* low — the architecture works without this; it's a calibration improvement, not a foundation. Ship and learn.

## Required behavior

New method on `forecast` activation:

```
forecast.resolve(
    program_id: String,
    actual: bool,               // did the predicted event happen?
    resolved_at: Option<String>,// ISO 8601; defaults to now
) -> Stream<ResolveEvent>
```

Behavior:

1. Read `programs/<program_id>/manifest.json` (must be `forecast.create` or `forecast.update`).
2. Read most recent `belief_state.md` for the prediction.
3. Append to `programs/_calibration/history.jsonl`: `{program_id, predicted, actual, deadline, resolved_at}`.
4. If `history.jsonl` has ≥ 10 entries, recompute Platt parameters and write to `programs/_calibration/bias.json`.

Subsequent `forecast.update` calls read `bias.json` and apply the calibration after aggregation but before writing the final state.

## Acceptance criteria

1. `forecast.resolve` registered.
2. Integration test: create 10 fake forecasts with known (predicted, actual) pairs designed to show systematic over-confidence; call `resolve` on each; assert `bias.json` is fit and that a subsequent `forecast.update` produces a probability shrunk toward 0.5 vs. the same call before resolution.
3. Integration test: with fewer than 10 resolved, `bias.json` is identity (or doesn't exist) and updates are passed through without calibration.
4. `cargo build` and `cargo test` pass green.

## Completion

Tests pass; status flips to `Complete` in the same commit. Epic MNEME-1's `## Calibration` section is filled in based on phase-1 outcomes.
