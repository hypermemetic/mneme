---
id: BLFX-5
title: "LOO-CV-tuned α aggregation (replaces fixed-prior shrinkage)"
status: Ready
type: implementation
blocked_by: []
unlocks: [BLFX-13]
confidence: medium
---

## Problem

Our current logit aggregation hardcodes `λ=0.2` toward `prior=0.5`, equivalent to `α=0.8` in the paper's notation. The paper learns `α` via leave-one-out cross-validation (§3 Multi-trial aggregation). On ForecastBench, paper finds α≈1 (no shrinkage). We systematically over-shrink.

## Context

The paper's formula: `p̂ = σ(α · (1/K) Σ logit(pₖ))` where `α ∈ [0, 1]` is estimated by LOO-CV that minimizes Brier on (predicted, actual) pairs.

Our current code in `mneme-substrate/src/mneme/swarm/aggregate/logit.rs` implements `sigmoid((1-λ) · trial_mean_logit + λ · prior_logit)`. With `prior_logit=0` (i.e., prior=0.5), this is mathematically `sigmoid((1-λ) · trial_mean_logit)` = paper's formula with `α = 1-λ`.

So the SHAPE is right; the ESTIMATION is wrong. We hardcode α; paper learns it.

## Evidence

The paper finds α≈1 on ForecastBench specifically. This means: for that benchmark + LLM combination, the optimal aggregation is just plain logit-space mean — no shrinkage. Our hardcoded α=0.8 systematically introduces ~20% pull toward 0.5 in logit space, which is directionally wrong.

But the paper notes this is "data-dependent" (different α optimal on different datasets / LLMs). So the right move isn't to set α=1 hardcoded; it's to estimate α from our calibration history.

Cold-start handling: until we have enough resolved forecasts to estimate α reliably (paper says ~30 minimum, similar to Platt's cold-start), use α=1 as the default (no shrinkage). This avoids the over-shrinking we currently do.

## Required behavior

```rust
pub fn estimate_alpha_loo(history: &[(Vec<f64>, bool)]) -> Option<f64>;
// Returns None if history.len() < ALPHA_COLD_START_THRESHOLD (30).
// Otherwise returns the alpha in [0, 1] that minimizes leave-one-out Brier.

pub fn logit_shrinkage_v2(
    trials: &[Value],
    field: &str,
    alpha: f64,
) -> Result<Value, AggregateError>;
// Same as current logit_shrinkage but with the paper's formulation.
```

Behavior:

| Scenario | Outcome |
|----------|---------|
| Calibration history has < 30 entries | Use α = 1.0 (no shrinkage), document as "cold start" in the artifact |
| History ≥ 30, LOO finds α=1 | Use α=1, document |
| History ≥ 30, LOO finds α<1 | Use the learned α, document |
| LOO fitter fails (numerical issue) | Fall back to α=1, log warning |
| Aggregation called with a hardcoded α (override) | Use that, ignore LOO |

The forecast activation reads the calibration store to fetch the current α before each `swarm.aggregate` call.

## Risks

| Risk | Mitigation |
|------|-----------|
| LOO fit overfits with 30 obs | Use simple grid search over α ∈ {0.0, 0.1, ..., 1.0} or constrained optimization with ridge; monitor stability across more obs |
| Calibration history is per-source but α is global | Phase 1: global α only. Per-source α is a future enhancement. |
| Existing `LogitShrinkage` rule callers break | Keep the old rule for backwards compat; add new `LogitAlpha` rule that takes α directly |

## What must NOT change

- The existing `AggregationRule::LogitShrinkage` variant (used by tests and old artifacts).
- The `MnemeStorage` schema beyond adding the calibration history columns the LOO fit needs.

## Acceptance criteria

1. New aggregation rule `AggregationRule::LogitAlpha { field, alpha }` — pure formula, no prior.
2. `estimate_alpha_loo(history)` function in `mneme/calibration/`, with unit tests covering: empty history, single-class history, well-mixed history, LOO grid sweep correctness.
3. `forecast.update`'s aggregation call switched to `LogitAlpha` with α from the calibration store (or 1.0 if cold-start).
4. The artifact records which α was used and whether it was cold-start vs learned.
5. Unit test: synthetic history where true optimal α is 0.5; assert `estimate_alpha_loo` returns within 0.1 of 0.5.
6. `cargo test --lib` passes green.

## Completion

Tests pass; status flips to `Complete` in the same commit.
