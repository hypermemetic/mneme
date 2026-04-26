---
id: BLFX-8
title: "Crowd signal injection for market questions"
status: Pending
type: implementation
blocked_by: []
unlocks: [BLFX-13]
confidence: low
---

## Problem

For market questions (Polymarket, Manifold, Metaculus), the paper injects the current market price as an anchor in the prompt. Paper §3 Crowd and empirical prior: *"For market questions, the crowd signal (market price) is injected into the prompt as an anchor; adding it substantially improves market BI (Table 4)."* This is significant because all top FB methods get the crowd estimate as a strong prior — so without it, we underperform on market questions specifically.

## Context

For backtesting, the "crowd signal" is the market price as of the cutoff date. For live forecasting, it's the current market price. Both need a market data source.

Current state: we don't have any market data integration. ForecastBench data (BLFX-10) likely includes the crowd signal per question per cutoff; if so, this ticket just threads it through the prompt.

## Evidence

This ticket is `low` confidence because it depends on having clean access to crowd signal data:
- For backtesting: the dataset must include `crowd_signal_at_cutoff` per question; if it doesn't, this ticket is much harder (requires per-source historical price scraping).
- For live: requires real-time API access to each market platform.

The implementation is straightforward IF the data is there: one prompt field. The risk is data availability.

## Required behavior

`forecast.update` accepts an optional `crowd_signal: Option<f64>` parameter (probability in [0, 1]). When provided, the prompt includes: *"Current market consensus for this question: p_market = X.XX"*. When None, no injection.

For backtesting: the backtest runner (BLFX-12) reads `crowd_signal_at_cutoff` from the dataset and passes it as the parameter.

| Scenario | Outcome |
|----------|---------|
| crowd_signal=Some(0.65) | Injected into prompt |
| crowd_signal=None | No injection |
| crowd_signal=Some(NaN) or out-of-range | Treated as None; logged warning |

## Risks

| Risk | Mitigation |
|------|-----------|
| ForecastBench data doesn't include crowd signals | Scrape from Polymarket/Manifold APIs at backtest time; cache aggressively |
| Crowd signal moves the LLM too much (anchoring bias) | Paper finds it's a net positive; trust the paper's empirical evidence |
| Live market data costs $$ | Live forecasting isn't this ticket's scope; just backtesting |

## What must NOT change

- forecast.update for callers who don't pass crowd_signal.

## Acceptance criteria

1. `forecast.update` accepts `crowd_signal: Option<f64>`.
2. Prompt template injects when provided.
3. Unit test: with crowd_signal=Some(0.7), assert the prompt sent to the trial contains "p_market = 0.70".
4. Integration test: live forecast.update against a mock with crowd_signal=Some(0.8); verify program inputs include crowd_signal.
5. `cargo test --lib` passes green.

## Completion

Tests pass; status flips to `Complete`.
