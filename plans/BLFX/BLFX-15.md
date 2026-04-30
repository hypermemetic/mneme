---
id: BLFX-15
title: "Source-specific data tools (yfinance, FRED, DBnomics, Wikipedia-as-of-date)"
status: Ready
type: implementation
blocked_by: [BLFX-4]
unlocks: [BLFX-13]
confidence: medium
forecast:
  hypothesis: "Will yfinance + FRED data tools (the first two of the four planned sources) ship within 21 days AND produce a paired Brier Index improvement of ≥ +3.0 BI on the quantitative-financial subset of bench-006 questions (subset = questions where the resolution criterion references a price, rate, index, or numeric macroeconomic indicator)?"
  resolution_method: "After yfinance + FRED tools land in agent_loop's execute_action and ClaudecodeStepDriver, manually tag each bench-006 question as quant-financial or not (one-pass YES/NO). Rerun forecastbench_holdout_run.py with these tools enabled. Resolves YES if (paired Δ BI on the quant subset with tools active) − (paired Δ BI on same subset bench-006 baseline) ≥ +3.0 AND the subset has ≥10 questions; NO if < +3.0 or the subset is too small to score; N/A if yfinance + FRED aren't both shipped by deadline. The risk is the model already getting these answers correctly via WebSearch, making the structured tools redundant."
  deadline: "2026-05-22T00:00:00Z"
---

## Problem

The paper's BLF apparatus exposes the LLM to four source-specific data
fetchers, in addition to generic web search:

- `fetch_ts_yfinance(ticker, range)` — Yahoo Finance time series.
- `fetch_ts_fred(series_id, range)` — Federal Reserve Economic Data.
- `fetch_ts_dbnomics(provider_code, dataset_code, series_code, range)` —
  DBnomics aggregated economic series.
- `fetch_wikipedia_section(article, section, as_of_date)` — Wikipedia
  *with revision-as-of-date* for date-leakage-safe history queries.

Today our `Action` enum (`mneme-substrate/src/activations/forecast/agent_loop.rs`)
declares two of these (`FetchTimeSeries { source, key }`,
`FetchWikipediaSection`) but `execute_action` returns
`Observation::Error("not implemented")` for both. They're stubs that exist
to make the action enum exhaustive, not actually wired.

## Why this isn't optional anymore

BLFX-1 explicitly punted source-specific tools as "out of scope: We rely on
Claude Code's generic Read + WebSearch + Bash for now." But:

- WebSearch is noisy for time-series queries — the model has to text-parse
  numbers out of news articles, with no guarantee of consistent source.
- Wikipedia-without-as-of-date is a date-leakage hazard for backtesting
  (BLFX-9 will mandate as-of-date for any historical query).
- Source-specific empirical priors (BLFX-7) needs a way to retrieve the
  actual time-series that defines the prior.

So this needs to land before BLFX-13 (faithful benchmark) is meaningful.

## Required behavior

Each fetcher becomes a real claudecode-side or substrate-side tool and is
reachable from inside an iterative trial via the `Action` enum:

| Action | Backend | Returns |
|---|---|---|
| `FetchTimeSeries { source: "yfinance", key: "AAPL", range }` | yfinance HTTP API or `yfinance` Rust crate | `Observation::TimeSeries { points }` |
| `FetchTimeSeries { source: "fred", key: "GDPC1", range }` | FRED public API (no key needed for limited use; key envvar otherwise) | same |
| `FetchTimeSeries { source: "dbnomics", key: "<provider>/<dataset>/<series>", range }` | DBnomics public API | same |
| `FetchWikipediaSection { article, section, as_of: DateTime<Utc> }` | Wikipedia REST API with revision lookup | `Observation::WikipediaSection { content }` |

Source `key` shape varies by source — encode in a struct or in the `key`
field's prose; details in implementation.

`as_of: Option<DateTime<Utc>>` extension to `FetchWikipediaSection`: if
`None`, current revision; if `Some(date)`, the latest revision whose
`rev_timestamp <= date`. This is the date-leakage primitive used by BLFX-9.

## Architecture choices

These tools live substrate-side, NOT inside the claudecode session, because:
- Substrate can guarantee deterministic source (no scraping ambiguity).
- Substrate can enforce the as-of-date contract for backtesting.
- The iterative loop already routes through `execute_action`, which is
  substrate-side.

The claudecode session's `WebSearch` tool stays as the "open-ended search"
escape hatch; the source-specific tools are the structured-data path.

## Acceptance criteria

1. `execute_action` returns real `Observation::TimeSeries` for FRED + yfinance
   queries (DBnomics can defer; pick whichever pair is fastest to wire).
2. `Action::FetchWikipediaSection` accepts an `as_of: Option<DateTime<Utc>>`
   field; substrate fetches the correct historical revision; integration test
   covers a Wikipedia page that's been edited recently and demonstrates
   different content for two different `as_of` values.
3. Live test: an iterative trial (BLFX-4 wired) emits a `fetch_ts_fred`
   action; the substrate executes it; the trial sees real series data and
   incorporates it into the next belief update.
4. Errors (rate limits, missing series, date out of range) return
   `Observation::Error { message }`; the loop continues without aborting.
5. `cargo test --lib` passes.

## Out of scope

- A unified data-source abstraction; source-specific clients are fine.
- Caching layer (separate ticket if rate limits become a problem during
  bench runs).
- DBnomics if FRED + yfinance are sufficient for benchmark coverage.

## Completion

Tests pass + live demonstration per (3); status flips to `Complete`.
