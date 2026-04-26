---
title: "BLFX-S01 — ForecastBench data access (spike)"
date: 2026-04-26
status: complete
---

## Headline

Dataset is **fully open and redistributable** (CC BY-SA 4.0), GitHub-hosted,
no auth needed. Backtest hygiene is built into the data shape (per-question
freeze datetimes + as-of values). Murphy 2026's "ForecastBench leaderboard"
references the same dataset Karger et al. published at ICLR 2025
(arXiv 2409.19839).

## Where the data lives

- **Code:** https://github.com/forecastingresearch/forecastbench (MIT)
- **Data:** https://github.com/forecastingresearch/forecastbench-datasets (CC BY-SA 4.0)
- **Site:** https://www.forecastbench.org
- **Benchmark paper:** Karger et al., ICLR 2025, arXiv 2409.19839 (cite alongside Murphy 2026)

Raw URL pattern:
- Questions: `datasets/question_sets/YYYY-MM-DD-llm.json`
- Resolutions: `datasets/resolution_sets/YYYY-MM-DD_resolution_set.json`

## Schema

**Question:**
```json
{
  "id": "<source-specific>",
  "source": "manifold|metaculus|polymarket|infer|acled|dbnomics|fred|wikipedia|yfinance",
  "question": "string (may contain {forecast_due_date} / {forecast_evaluation_date} f-string placeholders)",
  "resolution_criteria": "...",
  "background": "...",
  "url": "...",
  "freeze_datetime_value": "<state of underlying market/series at cutoff>",
  "freeze_datetime_value_explanation": "...",
  "market_info_open_datetime": "<ISO 8601>",
  "market_info_close_datetime": "<ISO 8601>",
  "market_info_resolution_datetime": "<ISO 8601>",
  "forecast_horizons": [7, 30, 90, 180, 365, 1825, 3650]  // empty for market questions
}
```

Top-level: `{forecast_due_date, question_set, questions[]}`. Each release ~500 questions.

**Resolution:**
```json
{
  "id": "...", "source": "...", "direction": "...",
  "resolution_date": "<ISO 8601>",
  "resolved_to": 0.0–1.0,   // probabilistic, not strict bool — use directly for Brier
  "resolved": true
}
```

Top-level: `{forecast_due_date, question_set, resolutions[]}`. Resolution sets are
partial per release; to score one prediction may need to union several
resolution_set files (or wait for the longest horizon).

## Backtest hygiene (the critical bit)

- `forecast_due_date` is the cutoff — pretend "today" = that date.
- `freeze_datetime_value` snapshots the underlying market/series state so you
  don't peek beyond the cutoff.
- ForecastBench's own paper claims **leakage rate < 1.5%** with this protocol;
  Murphy 2026 mirrors it.
- Workflow: pick a `(question_set, resolution_set)` pair where the resolution
  set's `forecast_due_date >= question_set.forecast_due_date + max(horizon)`.
  Feed only fields available at the cutoff to the model.

## Quirks worth knowing

- **f-string templating:** dataset-source `question` strings need
  `.format()` rendering before sending to the model. Don't ship raw braces.
- **Resolution sets are partial:** for a one-off score, you may need to
  union multiple resolution_set files.
- **`resolved_to` is probabilistic** — markets can resolve fractionally
  (e.g. 0.5 for ambiguous outcomes). Use raw float for Brier, threshold
  only when storing to our calibration store (which currently takes `bool`).
- **`latest-llm.json` is a 19-byte stub.** Don't use it; pin a real dated file.
- **Counts:** ~500 questions/release, biweekly since 2024 → 40+ releases,
  dozens fully resolved. Murphy's "400 backtest questions" = a single
  curated past release subset.

## Recommended ingestion workflow

One-shot vendored snapshot. Concretely:

1. **Sparse-checkout** of `datasets/question_sets/` + `datasets/resolution_sets/`
   from `forecastbench-datasets` into `programs/_benchmarks/forecastbench/`
   (gitignored — too large + already CC-licensed elsewhere).
2. **Pin a manifest** (`forecastbench.toml`) listing the question_sets +
   matching resolution_sets to evaluate against. Reruns are deterministic.
3. **Loader** is a Rust module that reads `(question_set, horizon) → ForecastQuestion`,
   renders the f-string, and emits a struct ready for `forecast.update`.
4. **Refresh:** nightly `git pull` on the submodule; never fetch per-question
   at runtime (rate limits + reproducibility hazard).
5. **Attribution:** include Karger et al. ICLR 2025 citation + CC BY-SA notice
   in the benchmark dir's README.

## Decisions for BLFX-10/11/12

- Use the GitHub repo directly (not Hugging Face mirror — single source of truth).
- Vendor via sparse-checkout, not fetch-on-demand.
- Brier scoring uses raw `resolved_to ∈ [0,1]`; threshold at 0.5 only when
  appending to our calibration store (which is `actual: bool`).
- First MVP run: **30 fully-resolved questions from 2024-Q3/Q4 release**
  (resolution sets exist; forecasts predate Sonnet's training cutoff so
  contamination concern still applies but we'll know if the pipeline works).
- Honest run: questions from 2026-Q1/Q2 with horizons resolving 2026-Q3+.
  Defer that to BLFX-13 once pipeline is verified on resolved data.

## Out of scope for BLFX-10/11/12

- Multi-horizon evaluation. First cut picks the *shortest* horizon per
  dataset question; market questions resolve at one fixed datetime anyway.
- Crowd-signal injection (BLFX-8) — orthogonal.
- Source-specific data tools (BLFX-15) — orthogonal.

## Sources

- https://github.com/forecastingresearch/forecastbench
- https://github.com/forecastingresearch/forecastbench-datasets
- https://www.forecastbench.org/datasets/
- https://github.com/forecastingresearch/forecastbench/wiki/How-does-ForecastBench-work%3F
- https://arxiv.org/abs/2409.19839 (canonical benchmark paper)
- https://arxiv.org/abs/2604.18576 (Murphy 2026 BLF)
