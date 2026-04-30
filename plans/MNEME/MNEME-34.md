---
id: MNEME-34
title: "Polymarket integration — real-money paired forecasting alongside Manifold"
status: Ready
type: implementation
blocked_by: [MNEME-30]
unlocks: []
confidence: medium
severity: Medium
forecast:
  hypothesis: "Will polymarket integration ship within 28 days AND, within 56 days of shipping, accumulate ≥30 paired (mneme, polymarket-crowd, actual) resolutions whose paired Brier delta is within ±0.03 of mneme's manifold-crowd paired delta from MNEME-30's first 30 manifold resolutions?"
  resolution_method: "After MNEME-34 ships, scripts/marketwatch_live.py begins firing forecasts on polymarket markets in addition to manifold's. Resolution sweep accumulates rows in resolutions.jsonl. After ≥30 polymarket resolutions land, compute mean(our_brier - poly_brier). Resolves YES if |mean(our_brier - poly_brier) - mean(our_brier - manifold_brier)| ≤ 0.03 AND ≥30 resolutions accumulated within 56 days post-ship; NO if delta gap > 0.03 or insufficient resolutions; N/A if MNEME-34 isn't built by ship deadline. Tests both the engineering (polymarket's API has auth + rate limits, harder than manifold's open API) AND whether mneme's edge is portable across crowd compositions (real-money polymarket should be more efficient than play-money manifold; substrate's signal-to-crowd ratio may be smaller)."
  deadline: "2026-07-30T00:00:00Z"
---

## Problem

Manifold (MNEME-30) is play-money — high-quality crowd but not as
efficient as real-money markets. Polymarket is the obvious
counterpoint: real money, deeper liquidity, more efficient pricing.

If our +26 BI delta on ForecastBench (with date defenses) and our
manifold paired delta are real, they should both translate — at lower
magnitude — to polymarket. If they don't, that's evidence we're
beating *play-money* crowd composition rather than the underlying
forecasting problem.

This is the third leg of the calibration story:
- ForecastBench (held-out): bench-006/008 — the algorithm number.
- Manifold live: MNEME-30 — out-of-sample, play-money crowd.
- Polymarket live (this ticket): out-of-sample, real-money crowd.

Combined, they tell us whether the substrate has a *durable* edge or
just a setup-specific one.

## Context

- **Polymarket's API** is at `https://gamma-api.polymarket.com/` and
  `https://clob.polymarket.com/`. Read-only market data is free; book
  data + trade history may need auth. Spec lives at
  https://docs.polymarket.com/.
- **Outcome handling**: polymarket binary markets resolve as
  YES/NO/INVALID. INVALID is rare but real (the sweeper must handle
  it like manifold's MKT/N/A — record but don't feed calibration).
- **Closing-time semantics**: `endDate` field is the deadline; some
  markets resolve before that.
- **Selection criteria** (mirrors manifold's): binary outcome, open,
  close in [14, 60] days, volume above some threshold.

## Required behavior

1. **Source abstraction.** Refactor `marketwatch_live.py` so the
   `fetch_open_binary_markets` + `fetch_market` calls dispatch on a
   `--source` flag (manifold | polymarket). Or split into
   `marketwatch_live_polymarket.py` if the API shapes diverge enough
   that abstraction is over-engineering.
2. **Polymarket API client.**
   - `fetch_open_binary_markets_polymarket()` — paginate gamma-api,
     filter to binary + open + closing in window.
   - `fetch_market_polymarket(market_id)` — single-market detail
     including current price + resolution state.
   - 404 / market-deletion handled the same as manifold (BLFX-9 /
     MNEME-33 patterns).
3. **Pairing rows in `pairings.jsonl`** carry `source: polymarket`
   (manifold rows already implicitly use manifold).
4. **Resolution sweep.** `marketwatch_resolve.py` already two-phase
   (Phase A network-only, Phase B substrate-feed); add polymarket
   resolution-checking to Phase A.
5. **Per-source paired analysis.** A `scripts/marketwatch_analyze.py`
   that splits pairings by source and reports per-source paired
   delta. Polymarket numbers vs manifold numbers should be
   side-by-side.
6. **Cron.** Same crontab pattern as manifold; can run as separate
   entries or a single combined invocation.

## Risks

| Risk | Mitigation |
|------|-----------|
| Polymarket auth required for the data we need | Start read-only; if needed, file a sub-ticket for auth flow (out of scope here) |
| Polymarket rate limits stricter than manifold's | Same `--max-pages` + concurrency knobs already applied to manifold |
| Polymarket markets are mostly US-political → narrow distribution that doesn't generalize | Document this caveat; per-source analysis surfaces it |
| Real-money efficiency means our paired delta is small/zero | That's an observable result, not a failure mode |

## What must NOT change

- Manifold path (MNEME-30) keeps working unchanged.
- Existing `pairings.jsonl` rows remain valid (manifold rows without
  an explicit `source` field are read as manifold).

## Acceptance criteria

1. `scripts/marketwatch_live.py --source polymarket` (or a sibling
   script) fires forecasts against polymarket markets and writes
   pairings with `source: polymarket`.
2. `scripts/marketwatch_resolve.py` sweeps polymarket resolutions in
   Phase A and feeds calibration in Phase B.
3. End-to-end smoke run: ≥3 polymarket forecasts fired, then manually
   verified in `pairings.jsonl`.
4. Per-source breakdown in any reporting tooling (`marketwatch_analyze.py`
   or equivalent) splits manifold and polymarket separately.
5. Documentation: `marketwatch_README.md` updated with polymarket
   usage + per-source caveats.

## Out of scope

- Real-money bet placement (the substrate observes; it does not bet).
  This stays out of scope until the substrate has an edge worth
  betting on, which is a much later ticket.
- Fee modeling. We compare raw probabilities, not net-of-fees outcomes.
- Custom polymarket auth flow if the open API is sufficient.

## Completion

`status: Complete` after acceptance criteria pass + a 7-day soak run
shows pairings accumulating without error in cron. The forecast
resolves later (need 30+ resolutions per the hypothesis) — completion
of the implementation does not require the forecast to resolve.
