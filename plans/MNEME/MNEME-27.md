---
id: MNEME-27
title: "Switch logit-shrinkage default from λ=0.2 to λ=0.0 (no shrinkage)"
status: Ready
type: implementation
blocked_by: []
unlocks: [BLFX-17, BLFX-18]
confidence: high
severity: Low
---

## Problem

`forecast.update`'s aggregation step in
`mneme-substrate/src/activations/forecast/activation.rs` calls
`AggregationRule::LogitShrinkage { field: "probability", prior: 0.5,
lambda: 0.2 }`. Hardcoded λ=0.2 shrinks every aggregated probability 20%
toward 0.5. Murphy 2026's empirical finding on ForecastBench was the
**opposite**: optimal α=1.0 (no shrinkage at all). So our hardcoded
default is biasing us toward 0.5 in the wrong direction — exactly the
direction that hurts confident-but-correct predictions.

bench-003's mneme score (BI 64.65 vs crowd 70.23) was measured with
λ=0.2 active. Setting λ=0.0 should mechanically improve the score
(typical effect: 1-3 BI on confident predictions). Without doing this,
later experiments (BLFX-17 iterative, BLFX-18 paired-n=100) measure
something that's already paper-misaligned.

This is a **one-line change** until BLFX-5 (LOO-CV-tuned α) lands.
After BLFX-5, the cold-start fallback should still default to λ=0.0
(matching the paper) rather than 0.2.

## Context

- The shrinkage call is in `run_update_in_background` in
  `mneme-substrate/src/activations/forecast/activation.rs`. Search for
  `LogitShrinkage` to find the exact line.
- The pipeline E2E test at
  `mneme-substrate/src/mneme/runtime/swarm_runtime.rs::pipeline_e2e_program_trial_aggregate_calibrate_artifact`
  uses λ=0.2 too — that test asserts the aggregate is between
  prior and trial mean. Loosening to λ=0.0 makes the aggregate equal
  the trial mean exactly; update the assertion accordingly.
- BLFX-5 (LOO-CV α) is filed Ready but not yet built. This ticket is
  the smaller "match the paper's empirical optimum *now*" change that
  doesn't wait for full LOO-CV machinery.

## Required behavior

1. Change `lambda: 0.2` → `lambda: 0.0` in `activation.rs::run_update_in_background`.
2. Update the pipeline E2E test's range assertion: with λ=0.0, the
   aggregated value equals the unweighted mean of trial logits, mapped
   back to probability space. Assert `aggregated == trial_logit_mean`
   (within 1e-9), not `prior < aggregated < trial_mean`.
3. Add an inline comment explaining why λ=0.0: "Per Murphy 2026 §4,
   optimal α=1.0 on ForecastBench (no shrinkage). BLFX-5 will replace
   this with LOO-CV-tuned α once enough resolved observations exist."
4. `cargo test --lib` passes.
5. `cargo build --bin mneme-substrate` clean.

## Acceptance criteria

1. The hardcoded λ in `activation.rs` is `0.0`.
2. The pipeline E2E test passes with the updated assertion.
3. All existing aggregation tests in
   `mneme-substrate/src/mneme/swarm/aggregate/logit.rs` still pass
   (those tests exercise multiple λ values explicitly; only the
   activation-level default changed).
4. `cargo test --lib` green.

## Out of scope

- Building LOO-CV α tuning (that's BLFX-5).
- Changing `LogitShrinkage`'s API or the underlying math.
- Re-running bench-003 — that comparison happens in BLFX-17/18 once
  the iterative loop is also being measured.

## Completion

Tests pass; commit message references this ticket and Murphy 2026 §4;
status → `Complete`.
