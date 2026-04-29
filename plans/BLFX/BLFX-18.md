---
id: BLFX-18
title: "bench-005: paired n=100 with paper-aligned config (α=1.0 + iterative + Platt)"
status: Complete
type: experiment
blocked_by: [BLFX-17, MNEME-26, MNEME-27, MNEME-28]
unlocks: [BLFX-13]
confidence: medium
---

## Problem

bench-003 has 95% CI on mean Brier of [0.004, 0.197], translating to a
BI CI spread of ~25 points. The 5.6 BI mneme-vs-crowd delta is well
inside that noise band. We cannot say from n=20 whether mneme is
meaningfully better or worse than the crowd, only that they are similar.

To produce **the first defensible mneme-vs-crowd number**, we need
n large enough to tighten the CI. n=100 is roughly the smallest sample
where a ±5 BI effect becomes detectable at p<0.05 on paired data. This
is the deciding bench — its result either justifies pushing into BLFX-13
(the integrated paper-comparison) or sends us back to fix something
algorithmic.

By the time this runs, the substrate's algorithm should match the
paper's empirical config (per its predecessor tickets):
- λ=0.0 (no shrinkage) per MNEME-27
- iterative_max_steps default per BLFX-17's outcome
- Platt calibration applied (post-MNEME-26 + MNEME-28)

## Context

- 188 joinable market questions in the 2024-07-21 release — 100 leaves
  88 untouched for an eventual held-out evaluation.
- Question selection: take 100 in resolution-date order (the script's
  default). This may correlate with question source (earlier-resolved
  questions skew toward shorter-term markets); record the source
  breakdown in the report.
- Wall-clock estimate: 188 questions takes ~50 min at concurrency=4
  with K=2 single-shot. At iterative T_max=5 it's ~5× the per-trial
  work, so ~3-4 hr wall-clock at concurrency=4 for n=100. Consider
  bumping concurrency=6-8 if the host can handle it (each container
  spawn is non-trivial RAM).
- Cost estimate: ~$25-50.

## Required behavior

**A. Run bench-005.** Same 2024-07-21 question set + resolution set as
bench-003/004, but n=100, K=2, with whatever
`iterative_max_steps` BLFX-17 defaulted to. Output to
`programs/_benchmarks/runs/<ts>-bench005-n100-paper-config/`.

**B. Run paired crowd baseline on the same 100.** Already inline in
`scripts/forecastbench_live_run.py` (`freeze_value_forecaster`).
Confirm the script logs both columns for the same 100 questions.

**C. Compute paired analysis.** Per-question Brier delta (mneme − crowd),
paired-bootstrap 95% CI on the mean delta and on delta BI. Critical:
the paired analysis is the headline number, not the independent-sample
BIs. Independent-sample BIs are reported but with appropriate caveats
(CIs are wide).

**D. Per-source breakdown.** Brier means + counts split by source
(manifold / metaculus / polymarket / infer). Both for mneme and crowd.
This pre-stages BLFX-6 (per-source Platt intercepts) — we'll know
which sources to target.

**E. Resolve into calibration store.** Each completed forecast should
flow into the calibration store via `forecast.resolve` so the n=100
result also seeds the data for BLFX-6/7. Reuse MNEME-26's resolver
script, parameterize over the run's `results.jsonl`.

**F. Compare to paper.** Paper's BLF claims ~+4 BI over the
ForecastBench crowd. Our crowd baseline measured here is ~70-72 BI;
"paper-faithful" target is mneme BI ≥ 73. Report whether we hit it.

## Acceptance criteria

1. Both runs (mneme + crowd-baseline) complete with ≤5 failures total.
2. `mneme/plans/BLFX/results/bench-005-n100-paper-config.md` committed.
3. The result doc reports:
   - Independent-sample BI for mneme + crowd, with bootstrap 95% CI
   - **Paired delta BI** with paired-bootstrap 95% CI (the headline)
   - Per-source breakdown table
   - Comparison to paper's claim
   - Disposition: do we hit "mneme ≥ crowd within statistical noise"?
   - Disposition: do we hit "mneme ≥ paper-claim BI 75 within
     statistical noise"?
4. Calibration store has ≥100 additional rows post-run.
5. The 88 untouched questions are noted as the held-out set for BLFX-13.

## Out of scope

- Multi-release union of resolution_sets (188 in this release is enough).
- Dataset-source questions (BLFX-15).
- Date-leakage defense (BLFX-9 — but if delta is suspect, mention as
  caveat in the report).

## Completion

Result doc committed; calibration store grew; disposition is clear.
Status → `Complete`. Calibration of BLFX-1 epic gets a one-line update.
