---
id: BLFX-10
title: "ForecastBench dataset ingestion"
status: Pending
type: implementation
blocked_by: [BLFX-S01]
unlocks: [BLFX-12]
confidence: medium
---

## Problem

To benchmark BLFX against the paper's reported numbers, we need the ForecastBench question + outcome dataset locally and in a queryable form. Paper used 400 questions across two tranches (A: 2025-10-26, B: 2025-11-09).

## Context

BLFX-S01 establishes how the data is accessible (download, scrape, or composite from underlying sources). This ticket implements the ingestion based on the spike's findings.

## Evidence

The ingestion is straightforward once the data path is known. The complication is that ForecastBench is a composite of multiple underlying sources (Polymarket, Manifold, Metaculus, RFI, yfinance, FRED, DBnomics, Wikipedia, ACLED). Paper's section H.2 documents the composition.

For each question we need:
- Question text
- Source + subtype
- Cutoff date
- Resolution date(s) — paper has multiple horizons per question
- Ground-truth outcome at each resolution date
- Crowd signal at cutoff (for market questions)
- Resolution URLs (for layer-4 URL blocking)

## Required behavior

```rust
pub struct ForecastBenchQuestion {
    pub id: String,
    pub source: String,                          // polymarket, manifold, metaculus, ...
    pub subtype: String,                         // binary, time-series-binarized, ...
    pub question: String,
    pub cutoff_date: DateTime<Utc>,
    pub resolutions: Vec<Resolution>,            // multiple horizons per question
    pub crowd_signal_at_cutoff: Option<f64>,
    pub blocked_urls: Vec<String>,
}

pub struct Resolution {
    pub date: DateTime<Utc>,
    pub outcome: bool,                           // resolved true/false
}

pub struct ForecastBenchDataset {
    questions: Vec<ForecastBenchQuestion>,
}

impl ForecastBenchDataset {
    pub fn load(path: &Path) -> Result<Self, Error>;
    pub fn iter(&self) -> impl Iterator<Item = &ForecastBenchQuestion>;
    pub fn filter_by_source(&self, source: &str) -> Vec<&ForecastBenchQuestion>;
    pub fn random_sample(&self, n: usize, seed: u64) -> Vec<&ForecastBenchQuestion>;
}
```

The dataset file lives at `mneme-substrate/data/forecastbench/{tranche-A,tranche-B}.json` (or similar). Loaded once at backtest startup.

## Risks

| Risk | Mitigation |
|------|-----------|
| BLFX-S01 returns a partial dataset (e.g., only market questions) | Document the gap; benchmark on what's available |
| Crowd signals not in dataset | Try to scrape; if not feasible, set to None and document the limit |
| Resolution outcomes have errors / disputed cases | Skip ambiguous questions per paper's methodology (§A.1) |
| Dataset is large enough to slow CI | Don't ship in the binary; download script + cached parquet |

## What must NOT change

- Anything else; this is purely additive ingestion.

## Acceptance criteria

1. `ForecastBenchDataset`, `ForecastBenchQuestion`, `Resolution` types.
2. Loader for the dataset format chosen in BLFX-S01.
3. At least 100 questions ingested with all required fields populated.
4. Unit tests: load test fixture, iterate, filter by source, random_sample reproducibility (same seed → same sample).
5. `cargo test --lib` passes green.

## Completion

Tests pass + the dataset file is committed (or a download script if too large); status flips to `Complete`.
