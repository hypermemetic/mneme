---
id: MNEME-25
title: "Concurrency parameter on the ForecastBench backtest runner"
status: Complete
type: implementation
blocked_by: [MNEME-24]
unlocks: [BLFX-13]
confidence: high
severity: Medium
---

## Problem

`benchmarks::forecastbench::runner::run_backtest` iterates questions
sequentially. With ~2 min per question and a target of ≥500 questions,
wall-clock is ~16 hours. Question-level parallelism (8 concurrent
questions) collapses it to ~2 hours.

## Required behavior

Add a `concurrency: usize` parameter (default `1` for backwards-compat).
Use `futures::stream::iter(...).buffer_unordered(concurrency)` to drive
up to N forecaster calls in flight simultaneously, collecting results
as they complete.

Determinism: results should still be sortable by question_id /
resolution_date so reports are reproducible. Don't rely on completion
order.

## Acceptance criteria

1. Existing tests still pass (concurrency=1 default).
2. New test: forecaster with artificial 200ms delay; n=8 questions at
   concurrency=4 completes in <500ms (4× speedup observable).
3. `cargo test --lib` passes.

## Out of scope

- Adaptive concurrency (start small, ramp up if errors low).
- Per-source rate limit awareness.

## Completion

Tests pass; status flips to `Complete`.
