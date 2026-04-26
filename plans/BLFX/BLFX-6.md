---
id: BLFX-6
title: "Hierarchical Platt with per-source intercept offsets"
status: Ready
type: implementation
blocked_by: []
unlocks: [BLFX-13]
confidence: medium
---

## Problem

Our Platt scaling is flat: one global `(a, b)` pair fit across all (predicted, actual) pairs. The paper uses hierarchical Platt: a global slope `a`, plus per-source intercept offsets `δ_s` (L2-regularized). The paper notes: *"global Platt scaling can over-shrink well-calibrated extreme predictions; hierarchical Platt with per-source intercept offsets ... outperforms global calibration in all settings."* Largest effect is on sources with skewed empirical priors.

## Context

The current Platt fit in `mneme-substrate/src/mneme/calibration/platt.rs::fit_platt` solves a 2-parameter Newton problem on the global pool. To go hierarchical, we add a per-source intercept offset `δ_s` such that the calibrated probability for source `s` is:

  `p_calibrated = σ(a · logit(p_raw) + b + δ_s)`

The fit minimizes Brier (or log-loss) over the pool, with L2 penalty `λ Σ δ_s²` to keep the offsets bounded when a source has few obs.

## Evidence

The paper's contribution (§§1, 3): hierarchical avoids the over-shrinking that occurs when sources have very different base rates (e.g., a source where 90% of questions resolve YES vs one where 10% do). With flat Platt, the global `b` averages across these and miscalibrates both.

The `ResolvedObservation` row already has a `program_id` field; we need a `source` field added. Sources for ForecastBench: `polymarket`, `manifold`, `metaculus`, `rfi`, `yfinance`, `fred`, `dbnomics`, `wikipedia`, `acled`. We capture which source each forecast was made about; the calibration store groups observations.

## Required behavior

```rust
pub struct HierarchicalPlattParams {
    pub a: f64,                                  // global slope
    pub b: f64,                                  // global intercept
    pub delta: HashMap<String, f64>,             // per-source offsets
    pub ridge: f64,                              // L2 regularization weight
}

pub fn fit_hierarchical_platt(
    observations: &[(f64, bool, String)],       // (predicted, actual, source)
    ridge: f64,
) -> Result<HierarchicalPlattParams, PlattError>;

pub fn platt_apply_hierarchical(
    params: &HierarchicalPlattParams,
    p_raw: f64,
    source: &str,
) -> Result<f64, PlattError>;
// If source not in delta map, treats δ_s = 0 (no offset).
```

Behavior:

| Scenario | Outcome |
|----------|---------|
| Fit with empty obs | Err NotEnoughObservations |
| Fit with single source | Equivalent to flat Platt; delta is {source → 0} |
| Fit with multiple sources, balanced | Ridge keeps δ_s small; (a, b) dominate |
| Fit with one source having skewed prior | Its δ_s becomes large; other sources unaffected |
| Apply with unknown source | Treat δ_s = 0 (back off to global) |

## Risks

| Risk | Mitigation |
|------|-----------|
| Optimization is more complex (>2 params) | Use coordinate descent: fix a, b, fit δ_s analytically per source; then re-fit a, b; alternate until convergence |
| Some sources have 0-1 observations | Ridge handles this; their δ_s stays near 0 |
| Storage migration | Add `source TEXT` column to `mneme_programs` and a calibration_observations table |

## What must NOT change

- The existing `PlattParams` (used for flat fit) and `fit_platt` / `platt_apply` functions; they stay as the global-only path.
- The cold-start threshold (10 observations); applies to hierarchical too.

## Acceptance criteria

1. `HierarchicalPlattParams`, `fit_hierarchical_platt`, `platt_apply_hierarchical` in `calibration/platt.rs`.
2. Migration: `ResolvedObservation` gains a `source: String` field; `MnemeStorage` accepts and indexes it.
3. Unit tests: synthetic data with 3 sources of differing skew; assert per-source δ_s is large for skewed sources, small for balanced ones; assert apply returns calibrated probabilities that improve Brier vs flat Platt on the synthetic set.
4. The existing flat-Platt tests still pass.
5. `cargo test --lib` passes green.

## Completion

Tests pass; status flips to `Complete` in the same commit.
