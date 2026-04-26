---
id: BLFX-11
title: "Scoring + stats framework (Brier Index + bootstrap CI)"
status: Pending
type: implementation
blocked_by: []
unlocks: [BLFX-12]
confidence: high
---

## Problem

To compare BLFX's performance to the paper's reported numbers, we need to compute the same metrics they use:
- **Brier Score (BS):** mean squared error of probability predictions.
- **Brier Index (BI):** `100 × (1 - √BS)`, paper's primary metric. Higher is better; 0.5-Brier scores 50.
- **Metaculus Score (MS):** Metaculus's published scoring rule.
- **Bootstrap confidence intervals** for paired comparisons (e.g., BLFX vs zero-shot baseline on the same question set).

## Context

Math is mechanical. The bootstrap is standard. The work is implementing it cleanly + a comparison framework.

## Evidence

`high` confidence because this is pure-Rust math against well-defined inputs. No LLM, no external dependencies beyond a stats library (`statrs` or hand-rolled).

## Required behavior

```rust
pub fn brier_score(predictions: &[(f64, bool)]) -> f64;
pub fn brier_index(predictions: &[(f64, bool)]) -> f64;
pub fn metaculus_baseline_score(predictions: &[(f64, bool)]) -> f64;

pub struct PairedComparison {
    pub method_a: String,
    pub method_b: String,
    pub mean_diff: f64,                          // mean(BI_A - BI_B) per question
    pub ci_lower: f64,
    pub ci_upper: f64,
    pub p_value: f64,                            // two-sided
    pub n: usize,
}

pub fn paired_bootstrap_ci(
    a_predictions: &[(f64, bool)],
    b_predictions: &[(f64, bool)],
    n_iterations: usize,                         // typical: 10_000
    confidence: f64,                             // typical: 0.95
    seed: u64,
) -> PairedComparison;
```

Behavior:

| Scenario | Outcome |
|----------|---------|
| Predictions = [] | Err InvalidInput |
| Predictions all true with p=1 | BS = 0, BI = 100 |
| Predictions all p=0.5 (uniform) | BI = 50 (independent of outcomes) |
| Paired bootstrap: A and B identical | mean_diff = 0, p_value ≈ 1 |
| Paired bootstrap: A consistently 0.05 BI better than B | p_value < 0.05 with reasonable n |

## Risks

| Risk | Mitigation |
|------|-----------|
| Bootstrap is slow at 10_000 iterations × many comparisons | Parallelize; precompute per-question BI once |
| Floating-point precision in BI computation | Use f64 throughout; clamp predictions to [eps, 1-eps] before sqrt |
| Comparison framework doesn't match paper's methodology exactly | Paper §G.2 references the methodology; follow that |

## What must NOT change

- Existing aggregation / calibration math.

## Acceptance criteria

1. `brier_score`, `brier_index`, `metaculus_baseline_score`, `paired_bootstrap_ci` in `mneme/scoring.rs` (new module).
2. Unit tests: known inputs → known outputs (sanity check against hand calculations or scipy).
3. Bootstrap test: deterministic seed; same input → same output (reproducibility).
4. `cargo test --lib` passes green.

## Completion

Tests pass; status flips to `Complete`.
