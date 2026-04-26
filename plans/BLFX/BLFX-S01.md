---
id: BLFX-S01
title: "Spike: ForecastBench dataset access"
status: Pending
type: spike
blocked_by: []
unlocks: [BLFX-10]
confidence: n/a
---

## Question

Can we obtain the ForecastBench question + ground-truth-outcome dataset at the granularity needed to backtest BLFX, and in a form usable from Rust without rate-limiting concerns?

## Setup

1. Visit `forecastbench.org/leaderboards` and identify how question sets are published.
2. Look for a public download (CSV, JSON, parquet, GitHub repo) of the 400 backtesting questions referenced in the paper.
3. If only a leaderboard exists, check whether the question text + cutoff date + resolution date + outcome are downloadable.
4. Examine the paper's section A.1 ("backtest tranches") for how the dataset was constructed: market questions vs dataset questions, source-specific dates, etc.

## Pass condition

We can download (or scrape, with rate limits respected) at least 100 questions with all of: question text, cutoff date, resolution date, source, and ground-truth outcome (true/false). The data fits in a single file ≤100MB.

## Evidence to record

- The data source (URL, repo, API endpoint).
- Whether ground-truth outcomes are public for ALL 400 questions or just a subset.
- Per-question metadata available (source, question type, market vs dataset, etc.).
- License / terms of use.

## Fail → next

If ForecastBench data isn't directly downloadable: scrape from the leaderboard pages OR use the underlying source data (Polymarket, Manifold, Metaculus, RFI, FRED, Wikipedia) directly with date-clamped queries.

## Fail → fallback

If neither of the above works: build a smaller benchmark from public Metaculus API + manual curation (50-100 questions). Document that we're benchmarking against a custom subset, not paper's exact set.
