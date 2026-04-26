---
title: "bench-002 — crowd baseline on ForecastBench 2024-07-21 release"
date: 2026-04-26
status: complete
---

## Headline

**Crowd-baseline Brier Index: 71.29** on 188 resolved market questions from
the 2024-07-21 release. Mean Brier 0.0718. 95% CI on mean Brier:
[0.0508, 0.0942].

This is the actual bar mneme needs to beat. Murphy 2026's BLF claims
~75 BI on ForecastBench — only ~4 BI points above the crowd we just
measured. **The window for a real "SOTA" claim is narrow and the CI is
wide enough that small improvements may not be statistically detectable
on a sample this size.**

## Method

- **Data:** vendored question_set (`2024-07-21-llm.json`, 1000 questions)
  and matching resolution_set (`2024-07-21_resolution_set.json`,
  7579 resolution rows) from forecastingresearch/forecastbench-datasets.
  CC BY-SA 4.0.
- **Scope:** Phase 1 — market sources only (manifold, metaculus,
  polymarket, infer). Dataset sources (acled/fred/yfinance/wikipedia/
  dbnomics) deferred until BLFX-15 (source-specific tools) lands.
- **Joined sample:** 188 market questions with resolutions in this
  release. Source breakdown:
  - polymarket: 59
  - manifold: 58
  - metaculus: 55
  - infer: 16
- **Resolution date range:** 2024-07-25 → 2026-04-25.
- **Base rate:** 50/188 YES = 26.6%.
- **Forecaster:** `freeze_value_forecaster` — predicts the market's own
  cutoff price (`freeze_datetime_value`) as the probability. This is the
  zero-effort "trust the crowd" baseline.
- **Scoring:** ForecastBench convention BI = 100 · (1 − mean_Brier / 0.25).
  Bootstrap CI: 1000 resamples, 95% percentile.

## Numbers

| Metric | Value |
|---|---|
| n | 188 |
| Failures | 0 |
| Mean Brier | 0.0718 |
| Brier Index | 71.29 |
| 95% CI on mean Brier | [0.0508, 0.0942] |
| Implied 95% CI on BI | ~[62.3, 79.7] |

## What this tells us

1. **The market is sharp.** Mean Brier of 0.072 means predictions are
   close to actual outcomes — better than I expected before computing.
2. **Tiny improvements are hard to detect.** The 95% CI on mean Brier
   spans 0.0434 (= 0.0942 - 0.0508), implying CI on BI of ~17 points.
   A "+5 BI improvement" from a system would not be reliably distinguishable
   from noise on n=188.
3. **Murphy 2026's claim makes sense relative to this.** A BLF system
   that hits ~75 BI would be ~4 above this crowd baseline — believable
   but inside the noise band of this sample.
4. **For mneme's own evaluation we need either:** larger n (union multiple
   resolution_sets to get ≥500 resolved questions) or a higher-effect
   target.

## Implications for BLFX-13 (faithful benchmark)

- Aim for **n ≥ 500 resolved market questions**. Union resolution_sets
  from 2024-07-21 through 2026-04-26 (or whichever subset has compatible
  freeze cutoffs).
- Always include the **crowd-baseline column** in the report. Comparing
  against 50/50 baseline (BI=0) is misleading; the real comparison is
  against the crowd.
- Plan for **paired-question analysis**: same question, crowd vs mneme,
  paired t-test or sign test on Brier deltas. Statistical power is much
  better for paired comparisons than independent samples.

## Implications for the paper claim

The paper's "SOTA on ForecastBench" likely means ~+4 BI above the crowd
baseline. That's a real result IF reproducible — but:

- Single paper, not replicated.
- Many of the 2024-Q3/Q4 questions were resolvable with information in
  the LLM's training data — search-augmented LLMs and the crowd are
  drawing from related sources, so the gap may be inflated.
- Paired analysis would be more honest than the BI delta.

What I'd want to see before believing "+4 BI" as a real, durable
improvement: same numbers on a held-out set of questions resolving
*after* the LLM's training cutoff. We can build that with mneme over
the next 3-6 months by resolving forecasts via `forecast.resolve`.

## Vendored data location

`mneme-substrate/programs/_benchmarks/forecastbench/` (gitignored).
Currently contains only the 2024-07-21 pair. To expand: download more
date-pairs from
`https://github.com/forecastingresearch/forecastbench-datasets/tree/main/datasets/`.
