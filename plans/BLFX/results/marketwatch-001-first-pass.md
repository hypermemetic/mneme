---
title: "marketwatch-001 — first live Manifold forecasts"
date: 2026-04-30
status: ongoing
---

## What this is

The first live forecasting pass against open Manifold markets. Unlike
ForecastBench (frozen historical data), these are markets that **have
not yet resolved**. We forecasted alongside the Manifold crowd at
2026-04-29; outcomes will accumulate over the coming weeks.

This is the only data path that produces unambiguous calibration
evidence — no web-search contamination is possible because the answers
don't exist yet.

## Setup

- Source: Manifold public API (`api.manifold.markets/v0`)
- Filter: open binary markets, close window 7-90 days, volume ≥ 50
  (Manifold play-money units), top 10 by volume
- Substrate: mneme-substrate:dev (lenient parser + MNEME-29 cleanup
  recovery deployed)
- Forecaster config: K=2 trials, T_max=5, λ=0.0, Platt on (a=1.23,
  b=+0.05)
- Pass executed at 2026-04-29 22:30 UTC; 9.8 min wall-clock at
  concurrency=4

## Result — 10 paired (manifold_p, our_p) tuples

| market_id | manifold | mneme | mneme_raw | Δ | direction |
|---|---|---|---|---|---|
| I9On8Elh | 0.942 | **0.368** | 0.383 | **−0.573** | mneme says crowd is way overconfident |
| RcNs5ch6 | 0.490 | **0.146** | 0.186 | **−0.344** | mneme says crowd should lean NO |
| ItRzuIds | 0.730 | 0.905 | 0.857 | **+0.175** | mneme says crowd should be more confident YES |
| RyZcsusL | 0.809 | 0.965 | 0.935 | **+0.156** | same direction |
| R885yIqc | 0.693 | 0.657 | 0.620 | −0.036 | tight agreement |
| CsclUcnR | 0.734 | 0.720 | 0.675 | −0.013 | tight agreement |
| QUOQIUPu | 0.882 | 0.912 | 0.865 | +0.030 | tight agreement |
| UtAp0z5P | 0.626 | 0.660 | 0.623 | +0.034 | tight agreement |
| ds2yC0gO | 0.894 | 0.905 | 0.857 | +0.011 | tight agreement |
| R2EsgyLd | 0.028 | 0.013 | 0.028 | −0.014 | tight agreement |

**Mean(mneme − manifold) on this pass: −0.058.**

## Observations

1. **6/10 are tight agreements** (|Δ| < 0.05). Where mneme and the
   crowd see the same evidence, they converge.
2. **4/10 are notable disagreements** (|Δ| ≥ 0.15). These are the
   information-rich questions — when they resolve we'll learn whether
   mneme or the crowd was right on each.
3. **Big-disagreement direction is mixed.** Two say crowd is too high,
   two say crowd is too low. So mneme isn't systematically biased
   toward one direction — at least on this small sample.
4. **Calibration loop is firing.** Every `our_p` differs from
   `our_raw_p` because Platt is being applied. The shifts are small
   (~0.04 on average) which is consistent with the n=114 calibration
   store fitting close to identity (a=1.23, b=+0.05).
5. **Mneme runs slightly lower than market on this slice** (mean Δ
   −0.058). Could be:
   - Genuine signal (mneme is more sober than play-money crowd)
   - Sample noise (n=10)
   - Platt artifact (the calibration was fit on bench-003+005 data
     which had a particular base rate)
   Will be tracked over time.

## Operational shape

- **Pairings file**: `programs/_marketwatch/pairings.jsonl` —
  append-only, one row per forecast. Each row has both `manifold_p`
  (snapshot at forecast time) and `our_p` (post-Platt).
- **Resolution sweeper**: `scripts/marketwatch_resolve.py` — checks
  Manifold for each watched market; on resolution, calls
  `forecast.resolve` against substrate and appends to
  `resolutions.jsonl`. First sweep at 2026-04-29 found 0 resolutions
  (expected — these markets just got their first forecast).
- **Cron suggestion**: every 4 hours run `marketwatch_live.py`
  (idempotent within thresholds), every 24 hours run
  `marketwatch_resolve.py`. See `scripts/marketwatch_README.md`.

## What we'll learn

- **Within days**: 1-3 of these 10 markets will resolve (the
  shorter-horizon ones with `closeTime` < 14 days). First real
  resolved-pair data.
- **Within weeks**: most of the 10 resolve; we get a real Brier vs
  Brier comparison on out-of-sample data.
- **Within months**: with more passes (continuous polling), we
  accumulate a few hundred resolved pairings. That dataset is the
  artifact that produces defensible product claims.

## Caveats

- **Manifold is play-money**, not Polymarket. The crowd is high-quality
  but not as efficient as a real-money market. Treat manifold_p as
  "smart-amateur consensus" not "efficient-market price." Polymarket
  integration is a future ticket.
- **The forecaster's prompt includes the current Manifold price.**
  This biases mneme toward agreement. For independent signal, consider
  a `--strip-crowd-anchor` flag that removes the price from the prompt.
- **Market selection is manual** (top 10 by volume in a window).
  Selection bias matters; document the filter for any reported
  results.

## Next moves

1. Set up cron (or just run weekly) to keep the pairings file growing.
2. After ~30-50 markets have resolved (probably 4-8 weeks), do a
   first calibration analysis: paired Brier deltas with bootstrap CI,
   per-question-source breakdown, big-disagreement-outcomes audit.
3. **Polymarket adapter** as a follow-up — same shape, different API
   client.
