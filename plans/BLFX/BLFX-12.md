---
id: BLFX-12
title: "Backtest runner"
status: Complete
type: implementation
blocked_by: [BLFX-10, BLFX-11]
unlocks: [BLFX-13]
confidence: medium
---

## Problem

We need a runner that takes a ForecastBench question set, runs BLFX (or any registered method) over it with proper date-clamping, collects predictions, and writes a results file the scoring framework can consume.

## Context

A backtest runner is a single binary that orchestrates: load dataset → for each question → invoke method with cutoff date → record prediction → save predictions file → compute scores → emit comparison report.

The runner doesn't replace the substrate; it USES the substrate. For each question, it calls `forecast.update` (or whatever method) via the same Plexus dispatch path used by the live system, but with backtest parameters set (cutoff_date, source, subtype, crowd_signal, blocked_urls all populated from the dataset).

## Evidence

This is orchestration, not algorithm. The challenge is integrating the existing pieces: dataset loader (BLFX-10), forecast.update with full BLFX features (BLFX-2..9), scoring (BLFX-11), and a results format that's diffable across runs.

## Required behavior

```rust
pub struct BacktestConfig {
    pub method: String,                          // forecast | crowd_only | zero_shot | ...
    pub dataset_path: PathBuf,
    pub n_questions: Option<usize>,              // sample N for cost-bounded runs
    pub seed: u64,
    pub k_trials: u8,
    pub max_steps: u8,                           // T_max
    pub model: String,                           // sonnet | opus | ...
    pub allowed_tools: Vec<String>,
    pub output_path: PathBuf,                    // results.jsonl
}

pub async fn run_backtest(config: BacktestConfig) -> Result<BacktestResults, Error>;

pub struct BacktestResults {
    pub config: BacktestConfig,
    pub predictions: Vec<BacktestPrediction>,
    pub started_at: DateTime<Utc>,
    pub finished_at: DateTime<Utc>,
    pub total_cost_usd_estimate: Option<f64>,
}

pub struct BacktestPrediction {
    pub question_id: String,
    pub source: String,
    pub subtype: String,
    pub predictions_per_horizon: Vec<(DateTime<Utc>, f64)>,
    pub outcomes_per_horizon: Vec<(DateTime<Utc>, bool)>,
    pub program_id: String,                      // for replay / inspection
    pub trial_count: u8,
    pub steps_taken: u8,
}
```

The runner is invoked from a new binary `src/bin/mneme-backtest.rs`. Output is a JSONL file with one BacktestPrediction per line. Comparison runs the scoring framework over the file.

| Scenario | Outcome |
|----------|---------|
| Method = "forecast" | Each question dispatched to forecast.update with full BLFX params |
| Method = "crowd_only" | No LLM call; record crowd_signal as the prediction |
| Method = "zero_shot" | Single chat with question + cutoff; no iterative loop |
| n_questions = Some(50) | Random sample (seeded) of 50; useful for cost-bounded validation |
| Mid-run failure on one question | Log + continue; record the failure in results; final report shows N attempted, M succeeded |

## Risks

| Risk | Mitigation |
|------|-----------|
| Long-running (potentially hours) | Stream progress events; checkpoint every 10 questions to allow resume |
| API rate limits | Configurable concurrency cap; back off on 429 |
| Cost overruns | BLFX-S03 estimate; --dry-run option that skips actual LLM calls and just enumerates work |
| Comparing against the paper requires identical question subsets | The dataset (BLFX-10) preserves question_ids from paper; runner records which were used |

## What must NOT change

- The substrate's dispatch path.
- The forecast.update method signature.

## Acceptance criteria

1. `mneme-substrate/src/bin/mneme-backtest.rs` binary.
2. End-to-end run on a 5-question subset of ForecastBench using `method=forecast`; produces a results.jsonl file with 5 predictions.
3. Resume: kill the runner mid-way; restart with same config; verify it picks up from the last checkpoint.
4. `--dry-run` enumerates without spending API.
5. Cost estimate emitted at end of run.
6. `cargo build --bin mneme-backtest` succeeds; `cargo test` passes green.

## Completion

Live test on at least 50 questions (cost-bounded by BLFX-S03); results file written; status flips to `Complete`.
