---
id: BLFX-9
title: "4-layer date-leakage defense"
status: Ready
type: implementation
blocked_by: []
unlocks: [BLFX-13]
confidence: medium
forecast:
  hypothesis: "Will building BLFX-9 and re-running bench-006 with the date-leakage defense active produce a paired delta vs crowd that lands within ±10 BI of Murphy 2026's claimed +4 BI (i.e., delta in [-6, +14])?"
  resolution_method: "Re-run scripts/forecastbench_holdout_run.py against the same 2026-03-15 questions with BLFX-9's leakage-defense prompt active. Compute paired Brier delta. Resolves YES if delta in [-6, +14] BI; NO if delta > +14 (suggesting our +26 BI win has substantive non-contamination signal); N/A if BLFX-9 isn't built within deadline."
  deadline: "2026-05-27T00:00:00Z"
---

## Problem

For valid backtesting, the agent must not see information from after the question's cutoff date. Otherwise the LLM "knows the answer" and the benchmark is meaningless. The paper implements a 4-layer defense and reports a residual leakage rate of <1.5% post-audit.

## Context

The paper's 4 layers (§§2 Backtesting validity, B for details):

1. **Search engine date filtering.** Pass cutoff date to the search engine so results are filtered server-side.
2. **LLM-based leak classifier on results.** A separate LLM call classifies each search result for whether it appears to contain post-cutoff information; flagged results are dropped.
3. **Data tool date clamping.** Time-series and lookup tools (yfinance, FRED, Wikipedia) accept a cutoff and only return data up to that date.
4. **URL blocking for resolution sources.** Specific URLs known to contain resolution information for a question (e.g., the prediction market's resolution page) are blocked.

Without these, the LLM via WebSearch will trivially find the answer to any backtest question whose resolution date has passed.

## Evidence

This is required infrastructure for benchmarking, not a feature for production forecasting. Without it, we can't claim our benchmark numbers are valid. Even with it, the paper's audit shows ~1.5% residual leakage — non-zero but acceptable.

The 4 layers each address a different leak vector. None individually suffice.

## Required behavior

The `EnvContext` (from BLFX-3) carries a `cutoff_date: Option<DateTime<Utc>>`. When set, every action's execution applies the relevant filter:

```rust
pub struct EnvContext {
    pub cutoff_date: Option<DateTime<Utc>>,
    pub blocked_urls: Vec<String>,                // for layer 4
    pub leak_classifier: Option<Arc<dyn LeakClassifier>>,
}

#[async_trait]
pub trait LeakClassifier: Send + Sync {
    async fn classify(&self, result: &SearchHit, cutoff: DateTime<Utc>) -> bool;  // true = leaked
}
```

Layer-by-layer:

| Layer | Implementation |
|-------|----------------|
| 1. Search engine date filtering | When `cutoff_date.is_some()`, pass `before:YYYY-MM-DD` to the WebSearch query (or whatever the tool's date-filter syntax is) |
| 2. LLM-based leak classifier | If `cutoff_date.is_some()` AND `leak_classifier.is_some()`, run each result through the classifier; drop flagged results |
| 3. Data tool date clamping | `FetchTimeSeries` truncates points to `≤ cutoff_date`; `FetchWikipediaSection` fetches the article revision as of cutoff, not current |
| 4. URL blocking | `LookupUrl` rejects URLs in `blocked_urls`; the per-question metadata (from BLFX-10) populates this list |

## Risks

| Risk | Mitigation |
|------|-----------|
| Search engine doesn't support date filters reliably | Use `before:` and verify in classifier (layer 2 catches what layer 1 misses) |
| LLM-based leak classifier itself uses post-cutoff knowledge | Paper accepts ~1.5% residual leakage; we should match or be transparent if worse |
| Wikipedia revision-as-of-date is hard to fetch | For phase 1: skip Wikipedia history; document as known limit; update later if needed |
| Per-question blocked-URL list missing for ForecastBench questions | Synthesize from question metadata (e.g., the Polymarket URL for a market question is its resolution URL) |

## What must NOT change

- Production forecasting (no cutoff_date) is unaffected — all layers no-op.
- `claudecode`'s WebSearch behavior outside the BLFX context.

## Acceptance criteria

1. `EnvContext` and `LeakClassifier` trait in `forecast/agent_loop.rs`.
2. Each of the 4 layers implemented in the corresponding `execute_action` arm.
3. Unit tests: each layer in isolation against mocked search results / data tools.
4. Integration test: a fake search result with date > cutoff is dropped by layer 2; a blocked URL is rejected by layer 4.
5. Audit-style test: run BLFX on 10 known backtest questions; manually inspect the conversation history for any post-cutoff information; report leakage rate.
6. `cargo test --lib` passes green.

## Completion

Tests pass + audit results recorded; status flips to `Complete`.
