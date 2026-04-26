---
id: BLFX-13
title: "Faithful benchmark + ablation sweep"
status: Pending
type: implementation
blocked_by: [BLFX-2, BLFX-3, BLFX-4, BLFX-5, BLFX-6, BLFX-7, BLFX-8, BLFX-9, BLFX-10, BLFX-11, BLFX-12]
unlocks: []
confidence: low
---

## Problem

The whole point of this epic. Run the faithful BLFX implementation on ForecastBench, compute Brier Index, compare to the paper's reported numbers. If we're within statistical CI of the paper's BLF, we've validated the implementation. If we're not, we know what specifically diverges and by how much.

Also reproduce the paper's ablation sweep (§4 / Table 2): how much does each component contribute? This both validates our implementation and gives us per-component understanding.

## Context

This ticket runs everything together. It depends on every other phase being complete. The work is execution + analysis, not new implementation.

## Evidence

`low` confidence because:
1. Many things have to be right at once.
2. We may discover late that a component's implementation diverges meaningfully (e.g., our LOO-CV α estimator finds a different α than the paper does, which would mean either our LOO is wrong or our trial diversity is different).
3. Running cost is real (BLFX-S03's estimate); we don't get many tries.

But the result is the result. Either we hit the paper's numbers within margin or we don't, and either way it's the truth.

## Required behavior

Two artifacts:

**A. Main benchmark.** Run BLFX with all features enabled on the full ForecastBench question set (or a defended subset per BLFX-S03's cost estimate). Compute:
- Mean BI, broken down by source and overall
- Bootstrap 95% CI on the mean BI
- Comparison to the paper's reported BI for BLF (paper says ABI=71.0, primary table 1)
- Comparison to GPT-5 zero-shot, Cassi, Grok 4.20, Foresight-32B reported numbers (reproduce paper's table 1 with our column added)

**B. Ablation sweep.** Run with each component disabled in turn:
- BLFX-no-belief-state (collapse to {p, summary})
- BLFX-no-iterative-loop (single chat per trial)
- BLFX-no-search (allowed_tools = [])
- BLFX-no-hierarchical-platt (flat Platt)
- BLFX-no-α-tuning (hardcoded α=0.8)
- BLFX-no-empirical-prior
- BLFX-no-crowd-signal

For each ablation, compute mean BI and the paired-bootstrap CI vs the full system. Reproduce paper's Table 2 with our deltas.

## Risks

| Risk | Mitigation |
|------|-----------|
| Cost overruns on full benchmark | Run subset first (50 questions per BLFX-S03 fallback); if numbers look right, expand |
| Numbers don't match paper | Diagnose per ablation: which component contributes the gap? Treat as research finding, not failure |
| Paper's numbers were on Gemini-3.1-Pro; we use Sonnet; not directly comparable | BLFX-S04 documents the substitution; we report numbers as "BLFX-on-Sonnet" |
| Days of compute time on a single benchmark run | Parallelize across questions (concurrency cap from BLFX-12) |

## What must NOT change

- The implementations under test. This is observation, not modification.

## Acceptance criteria

1. `mneme/plans/BLFX/results/main-benchmark-{date}.json` — full benchmark results.
2. `mneme/plans/BLFX/results/ablation-{component}-{date}.json` — one file per ablation.
3. `mneme/plans/BLFX/results/REPORT.md` — written analysis: our numbers vs paper's, per-source breakdown, ablation deltas, honest discussion of where we match / diverge / can't compare.
4. Methodology section in REPORT.md: which question subset, which model, which random seed, total cost.
5. Final disposition: either (a) our numbers match paper within CI, validate the implementation; or (b) they don't, and the report explains which divergence is responsible.
6. The REPORT.md is the artifact this whole epic produces.

## Completion

REPORT.md committed; status flips to `Complete`. Epic MNEME-1 / BLFX-1's calibration sections both get a one-line note based on findings.
