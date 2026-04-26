---
title: "bench-003 — first real mneme-vs-crowd numbers (n=20, single-shot K=2)"
date: 2026-04-26
status: complete
---

## Headline

**Mneme is currently 5.58 Brier Index points WORSE than the crowd
baseline on a paired n=20 sample from ForecastBench 2024-07-21.**

| | Mean Brier | Brier Index | 95% CI on mean Brier |
|---|---|---|---|
| Mneme (single-shot K=2) | 0.0884 | **64.65** | [0.004, 0.197] |
| Crowd (freeze_value)    | 0.0744 | **70.23** | [0.027, 0.127] |
| Delta (mneme − crowd)   | +0.0140 | **−5.58** | — |

The crowd CI is narrower (n=20 paired). Both BIs are well above the
coin-flip baseline (BI=0); both are below "perfect" (BI=100). They are
in the same ballpark but the crowd wins on this sample.

Murphy 2026 claims their BLF beats the ForecastBench crowd by ~4 BI.
Mneme as currently scoped (single-shot, fixed shrinkage, no calibration
data, no source priors, no iterative loop) is going the opposite
direction by ~5.6 BI. **The gap from "where mneme is" to "where the
paper claims" is roughly ~10 BI on this benchmark.**

## What ran

- **Substrate:** mneme-substrate at commit 540dcae (MNEME-24 + MNEME-25
  parallel trials and concurrency wiring landed).
- **Mode:** single-shot per trial (`iterative_max_steps: None`), K=2
  trials in parallel.
- **Model:** Claude Sonnet (substrate's default).
- **Tools:** WebSearch only.
- **Forecaster prompt:** rendered question text + resolution criteria
  + source name + freeze_datetime_value, with a "forecast the probability"
  closing instruction.
- **Aggregation:** logit-shrinkage λ=0.2 toward prior 0.5 (mneme's hardcoded
  default; not LOO-CV-tuned per BLFX-5).
- **Calibration:** identity (cold-start; no resolved data in store).
- **Paired comparison:** for each of the same 20 questions, the crowd
  baseline used the question's `freeze_datetime_value` directly as its
  prediction.

## Wall clock + throughput

- **Total: 331 s (5.5 min) for 20 questions at concurrency=4.**
- Mean per question: 16.5 s.
- Range per question: 18 s – 153 s.

This validates MNEME-24 (parallel trials inside `swarm.trial`): K=2
completed in roughly K=1 wall clock. And MNEME-25 (concurrency on the
backtest runner): 4 questions in flight simultaneously.

Implication: a full 188-question run takes ~50 min. A 500-question run
(BLFX-13 target n) takes ~2 hr 10 min. Real, fast.

## Per-question breakdown — the misses dominate

3 of 20 questions had Brier > 0.3 — these dominated the mean:

| id | source | predicted | actual | Brier |
|---|---|---|---|---|
| 15796 | metaculus | 0.150 | 1.0 | 0.722 |
| xv86CDBe0flxF2epvO3f | manifold | 0.774 | 0.0 | 0.599 |
| 0x0590ee... | polymarket | 0.617 | 0.0 | 0.381 |

The other 17 had Brier ≤ 0.013 — mostly excellent.

**Question 15796** ("Will Russian athletes be barred from competing at
the 2024 Olympics?") is a textbook resolution-criteria-ambiguity
failure. The model reasoned: "15 Russian athletes competed under
neutral status, so technically they weren't barred — predict NO." But
the metaculus question resolved YES — the resolver applied a stricter
interpretation. The crowd had freeze_value=0.75 (correctly anticipating
the resolver's interpretation).

This is the exact failure mode the paper's hierarchical Platt
calibration with per-source intercepts (BLFX-6) is designed to address:
metaculus questions systematically resolve different from how the model
reads them; an intercept term for source=metaculus would shift mneme's
predictions toward the resolver's pattern.

## What this tells us about the work ahead

To close the 10 BI gap from "current mneme" to "paper claims":

1. **Iterative loop wired live (BLFX-4 + BLFX-16).** Already implemented
   but not yet benchmarked end-to-end. Murphy's ablation: +3.8 BI.
   Next: re-run this same n=20 with `iterative_max_steps: Some(10)` for
   a within-paper ablation.
2. **Hierarchical Platt with per-source intercepts (BLFX-6).** Requires
   resolved data — `forecast.resolve` is wired (MNEME-23) but the
   calibration store is empty until we record outcomes. This bench's 20
   resolved questions could seed it.
3. **LOO-CV-tuned α (BLFX-5).** Replaces hardcoded λ=0.2 with α tuned
   on resolved history. Murphy: optimal α=1.0 on ForecastBench
   (= no shrinkage). Hardcoded λ=0.2 is biasing predictions toward
   0.5 — for confident model calls (most of mneme's 17 good predictions)
   this hurts.
4. **More trials (K=5+).** Doesn't help directly but lowers variance
   per question. Cost is bounded by wall-clock now that trials run in
   parallel.

## What this DOESN'T tell us

- **n=20 is too small.** 95% CI on mneme's mean Brier spans [0.004, 0.197].
  That's a CI of ±~25 BI. The 5.6 BI delta is well inside this noise band.
  We cannot conclude mneme is meaningfully different from the crowd on
  this sample.
- **2024-Q3 questions had likely contamination.** Sonnet's training
  cutoff is well after July 2024; the model knew which Russian athletes
  competed at Paris 2024 from training data. Crowd had no such advantage
  (their freeze price is from before the events). On post-cutoff
  questions the gap may shift dramatically in either direction.

## Cost

Subjective estimate: ~$1-3 on Sonnet for this run. Token capture is
partial (MNEME-22 still open) so I can't quote exact USD. Per-question
average num_turns from the few we have visibility on: ~5-9 turns/trial
× 2 trials × 20 questions ≈ 200-400 turns total.

## Reproducibility

```bash
# In mneme-substrate/, with substrate running on port 4456:
python3 programs/_benchmarks/run_live_bench.py \
    --question-set programs/_benchmarks/forecastbench/2024-07-21-llm.json \
    --resolution-set programs/_benchmarks/forecastbench/2024-07-21_resolution_set.json \
    --n 20 --concurrency 4 --port 4456 --trials 2 \
    --output programs/_benchmarks/runs/<timestamp>/
```

Raw results at
`mneme-substrate/programs/_benchmarks/runs/20260426-165255-n20-k2/`
(gitignored; on local machine only).

## Recommended next moves (in order)

1. **Resolve all 20 questions into the calibration store** via 20 calls
   to `forecast.resolve`. Seeds organizational calibration data; we'll
   need this for BLFX-6 anyway. ~1 minute of work.
2. **Re-run identical bench with iterative_max_steps=5**, see if
   iterative beats single-shot on the same questions. Same ~5-7 min wall
   clock; ~2× cost. This tests Murphy's main ablation in our own setup.
3. **Scale to n=100 single-shot** (~30 min wall clock). Tightens CI on
   the mneme-vs-crowd delta enough to actually decide if mneme is
   reliably better/worse than the crowd.
4. **Then BLFX-13 full run on n=188 with both modes** for an honest
   comparable BI number. ~1.5 hours total.
