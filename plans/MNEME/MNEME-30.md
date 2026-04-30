---
id: MNEME-30
title: "Live marketplace pipeline (Manifold) — continuous forecast + pairing + resolution"
status: Complete
type: implementation
blocked_by: []
unlocks: []
confidence: medium
severity: Medium
---

## Problem

ForecastBench is the public benchmark; bench-006 / bench-007 give us
held-out post-cutoff numbers. But all of those sit in a frozen-history
mode — questions whose freeze date is in the past. The real test of a
forecasting system is on **markets that are currently open**, where:

1. No web search can retrieve the answer (it doesn't exist yet).
2. The market price moves — we can re-forecast on movement and watch
   convergence.
3. Eventually it resolves; calibration data accumulates as actual
   ground truth.

This is the load-bearing path to defensible product claims. ForecastBench
proves the algorithm; live forecasting proves the product.

## Context

- **Manifold Markets** has a free, no-auth public API at
  `https://api.manifold.markets/v0/`.
  - `GET /markets?limit=N` — paginated list with sort options
  - `GET /market/{id}` — full detail for one market
  - Each market has `outcomeType` (filter to `BINARY`), `isResolved`,
    `resolutionTime`, `resolution` (YES/NO), `probability` (current
    crowd estimate), `closeTime`, `volume`, `question`, `description`
  - Manifold uses play-money, which is fine for our purposes — we want
    the crowd-belief signal and the eventual binary resolution.
- **Polymarket** is real-money, more liquid, but harder to integrate
  (auth, rate limits). Defer to a follow-up.
- The substrate's existing `forecast.update` + `forecast.resolve`
  surface is sufficient — no new substrate-side methods. This ticket
  is mostly Python tooling on top.

## Required behavior

Two scripts plus a small data store on disk:

**1. `scripts/marketwatch_live.py`** — single-pass forecaster.

Operation:
1. Hits Manifold's API for binary markets matching a filter:
   - `outcomeType == "BINARY"`
   - `isResolved == false`
   - `closeTime` in next 14-60 days (configurable)
   - `volume >= threshold` (configurable, default 100)
2. For each selected market, checks the local pairings log to see
   if we've already forecasted recently:
   - First pass on a market → fire a forecast
   - Subsequent pass → fire only if `|current_price - last_forecast_price| > price_delta_threshold`
     OR `time_since_last > time_threshold` (e.g., 24 hours)
3. For each market that needs a fresh forecast, fires `forecast.update`
   via synapse with the question + description + current crowd price
   as evidence.
4. Records the pairing in `programs/_marketwatch/pairings.jsonl`:
   ```json
   {
     "ts": "2026-04-30T01:30:00Z",
     "market_id": "abc123",
     "market_url": "https://manifold.markets/...",
     "question": "Will X happen by Y?",
     "close_time": "2026-05-15T...",
     "manifold_p": 0.34,
     "our_p": 0.42,
     "our_raw_p": 0.50,
     "program_id": "<from-substrate>",
     "volume": 1234.5
   }
   ```
5. Designed to be re-run (cron, launchctl, or just manual) — idempotent
   on repeated invocations within the time/delta thresholds.

**2. `scripts/marketwatch_resolve.py`** — resolution sweeper.

Operation:
1. Reads `programs/_marketwatch/pairings.jsonl`.
2. Groups by market_id; for each market, finds all forecasts we made.
3. Hits Manifold API to check if the market has resolved.
4. If resolved YES/NO and we haven't recorded the resolution yet:
   - Append to `programs/_marketwatch/resolutions.jsonl` with the
     resolved outcome.
   - For each forecast we made on that market, call `forecast.resolve`
     against substrate so the calibration store grows.
5. Idempotent on repeated invocations (skips already-resolved markets).

**3. Operational notes** in
`mneme-substrate/scripts/marketwatch_README.md` — how to run, how to
schedule via cron, how to filter, what the data files mean.

## Acceptance criteria

1. `scripts/marketwatch_live.py` runs end-to-end against the live
   substrate + Manifold API. No auth tokens needed; no surprises on
   first invocation.
2. After one run, `programs/_marketwatch/pairings.jsonl` has ≥5 entries
   from real Manifold markets resolving in 14-60 days. Each entry
   contains both `manifold_p` (crowd) and `our_p` (mneme).
3. `scripts/marketwatch_resolve.py` runs without error against an
   empty resolutions file (no markets resolved yet — that's expected).
4. The pairings + resolutions files are gitignored (per-run state, may
   contain a lot of data over time).
5. README explains the manual operation + a cron suggestion for
   continuous monitoring (e.g., `every 4 hours, run marketwatch_live;
   every day, run marketwatch_resolve`).
6. After enough markets resolve (over weeks), the calibration store
   should grow with real out-of-sample observations. We don't verify
   this in the ticket — it's the long-term effect.

## Out of scope

- Polymarket integration (separate ticket).
- Real-money betting (we observe only).
- A web dashboard — pairings.jsonl + a small analysis script is
  enough for now.
- Streaming WebSocket polling — Manifold's REST API at moderate
  intervals (every few hours) is sufficient.
- Multi-marketplace pairing (correlate Manifold + Polymarket on the
  same underlying question). Future.

## Completion

Both scripts shipped + at least one initial pass run + result doc with
the first ≥5 paired (manifold_p, our_p) data points. Status →
`Complete`.
