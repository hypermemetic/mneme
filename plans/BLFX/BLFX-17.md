---
id: BLFX-17
title: "bench-004: paired iterative-loop vs single-shot on bench-003's n=20"
status: Complete
type: experiment
blocked_by: [BLFX-4, BLFX-16]
unlocks: [BLFX-18]
confidence: medium
---

## Problem

The iterative agent loop is the single biggest non-search algorithmic
component in Murphy 2026 (-3.8 BI when removed, per the paper's
ablation). We have it wired live (BLFX-4 scaffold + BLFX-16 real
WebSearch via search-worker session) but have never benchmarked it
end-to-end against single-shot on real ForecastBench data. bench-003 was
single-shot only, scored BI 64.65 on n=20 (5.6 BI below the crowd
baseline 70.23). If iterative truly buys us +3.8 BI, that closes most
of the gap to the crowd in one experiment.

This is the highest-value single experiment we can run today — paired
analysis on the same 20 questions has much better statistical power
than independent samples, so even a small effect is detectable.

## Context

- bench-003 results at
  `mneme-substrate/programs/_benchmarks/runs/20260426-165255-n20-k2/`.
  Same 20 questions, K=2 trials, single-shot, λ=0.2.
- Holding everything constant except `iterative_max_steps` ⇒ the
  measured delta is the iterative-loop effect.
- MNEME-27 (λ=0.0 default) lands in the same direction as iterative —
  bench-004 should run AFTER MNEME-27 so we're measuring iterative
  against the corrected baseline, not against a known-misaligned one.
  Re-run bench-003 first to get a fresh single-shot baseline at λ=0.0.
- Cost estimate: iterative T_max=5 K=2 = ~5-10× single-shot tokens, so
  ~10-15 min wall clock at concurrency=4, ~$5-15.

## Required behavior

Run two paired benches against the same 20 questions
(2024-07-21 release, the first 20 in resolution-date order — which is
exactly what bench-003 targeted by virtue of `take(20)` ordering):

**A. bench-004a — single-shot baseline at λ=0.0.** Prerequisite: MNEME-27
landed and substrate restarted. Same args as bench-003: K=2,
single-shot, concurrency=4. Output to
`programs/_benchmarks/runs/<ts>-bench004a-singleshot-lambda0/`.

**B. bench-004b — iterative loop.** Same 20 questions, K=2,
`iterative_max_steps=5`, concurrency=4. Output to
`programs/_benchmarks/runs/<ts>-bench004b-iterative/`.

Then a paired analysis — for each question, compute Brier under each
mode; bootstrap CI on the per-question Brier delta (paired). Report
**delta BI** (iterative − single_shot) with paired-bootstrap 95% CI.

If delta BI ≥ +2 with CI excluding 0: iterative wins decisively, mark
the iterative path validated, default `iterative_max_steps` to `Some(5)`
in `forecast.update`.
If delta BI ∈ [-2, +2]: iterative is in the noise; **do not** flip the
default — re-run at higher n to disambiguate.
If delta BI ≤ -2 with CI excluding 0: iterative is hurting in our setup.
Diagnose: is the per-step prompt still wrong? Are the search-worker
results being properly fed back into the reasoning session?

## Acceptance criteria

1. Both bench runs complete with 0 failures (any failure invalidates
   the comparison).
2. `mneme/plans/BLFX/results/bench-004-iterative-vs-single-shot.md`
   committed, containing:
   - The same headline-table format as bench-003
   - Paired Brier delta per question (table)
   - Paired-bootstrap 95% CI on the mean delta + delta BI
   - At least 3 example trial transcripts (one where iterative
     improves, one where iterative hurts, one where the model used the
     full T_max without converging) — drawn from
     `programs/<id>/sessions/` files
   - Disposition: which decision branch (above) we're taking
3. If decision is "iterative wins decisively": one-line code change to
   default `iterative_max_steps` lands in the same commit as the
   results doc.

## Out of scope

- Tuning `T_max` (pinned at 5 for this experiment; 10 is the paper's
  default but costs 2× more — defer the T_max sweep to a follow-up).
- Comparing on dataset-source questions (BLFX-15 territory — markets only).
- Paper-claim comparison (BLFX-13's job — this ticket only proves
  iterative helps in our setup).

## Completion

Result doc + decision committed. Status → `Complete`. If "iterative wins
decisively" branch taken, the default-flip commit references this
ticket as justification.
