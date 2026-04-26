---
id: BLFX-1
title: "BLF-Exact — faithful re-implementation of Murphy 2026 BLF on mneme"
status: Epic
type: epic
blocked_by: []
unlocks: []
---

## Goal

A faithful re-implementation of the BLF (Bayesian Linguistic Forecaster) algorithm from [Murphy 2026, arXiv 2604.18576](https://arxiv.org/abs/2604.18576), running on the mneme substrate, that we can backtest on ForecastBench and report Brier Index numbers comparable to the paper's.

End state: `forecast.update` produces forecasts that match (within statistical noise) the paper's reported BLF performance on ForecastBench. We can defend the SOTA claim with our own numbers.

## Why this epic

After reading the paper carefully (§§1–3), our current `forecast.update` is **BLF-shaped but not BLF-faithful**. The shape (linguistic belief state + multi-trial logit aggregation + Platt calibration) is right; the algorithm is approximated. Specifically, our implementation is missing:

- **The iterative agent loop.** The paper's biggest non-search contribution. Single-shot reasoning is ~3.8 BI worse on ablation.
- **Structured belief state.** Paper uses `{p, confidence, evidence_for, evidence_against, open_questions}`; we use `{p, summary}`. Worth ~3.0 BI.
- **LOO-CV-tuned α** instead of hardcoded shrinkage (paper finds α=1 on ForecastBench; we hardcoded α=0.8).
- **Hierarchical Platt** with per-source intercept offsets.
- **Source-specific empirical priors and crowd signal injection.**
- **4-layer date-leakage defense** (necessary for valid backtesting).

Until these land, "SOTA" should not appear in our docs.

## Architecture impact

The iterative agent loop is the load-bearing change. It changes `swarm.trial`'s shape:
- Today: each trial = one `claudecode.chat` call → parse final response.
- Faithful: each trial = a loop of `(action, belief) ← LLM(history); execute action; append result`, terminating on `submit` or T_max=10.

The other changes are localized:
- Aggregation lives in `mneme/swarm/aggregate/logit.rs` — replace fixed λ with LOO-tuned α.
- Calibration lives in `mneme/calibration/platt.rs` — extend with per-source intercepts.
- New modules for empirical priors, crowd signal, leak defense.

## Dependency DAG

```
BLFX-S01 (ForecastBench access spike)
   │
   └──> BLFX-10 (dataset ingestion)
          │
          └──> BLFX-12 (backtest runner) ──> BLFX-13 (faithful benchmark)
                 ▲
BLFX-S02 (action format spike) ──> BLFX-3 (action enum + parser) ──> BLFX-4 (iterative loop)
                                         ▲                                  │
BLFX-2 (structured belief state) ────────┴──────────────────────────────────┤
                                                                            │
BLFX-S04 (base LLM choice spike) ──> BLFX-4                                 ▼
                                                              (BLFX-13 needs everything)
BLFX-5  (LOO-CV α aggregation)         ─────┐
BLFX-6  (hierarchical Platt)            ────┤
BLFX-7  (source-specific empirical prior) ──┤
BLFX-8  (crowd signal injection)        ────┼────────────────────────────> BLFX-13
BLFX-9  (4-layer date leakage defense)  ────┤
BLFX-11 (scoring + stats framework)     ────┤
BLFX-15 (source-specific data tools)    ────┘  (depends on BLFX-4)

BLFX-S03 (cost estimate spike) is informational, gates whether BLFX-13 is feasible
```

## Substrate-side prerequisites (parallel to BLFX work)

| Ticket | Why it matters here |
|---|---|
| MNEME-22 | Token capture upstream — we can't honestly report cost-per-bench without it |
| MNEME-23 | `forecast.resolve` — without outcome recording, BLFX-6 (hierarchical Platt) has no data to fit on |

## Phases

### Phase 1 — Core algorithm (highest-impact divergences)

| Ticket | Title | Confidence |
|--------|-------|-----------|
| BLFX-S02 | Spike: action format — Claude tool-use vs custom JSON | n/a |
| BLFX-2 | Structured belief state schema and prompt | high |
| BLFX-3 | Action enum + parser (web_search, summarize_results, lookup_url, submit, ...) | medium |
| BLFX-4 | Iterative agent loop (per-trial multi-step belief update) | low |

### Phase 2 — Aggregation correctness

| Ticket | Title | Confidence |
|--------|-------|-----------|
| BLFX-5 | LOO-CV-tuned α aggregation (replaces fixed-prior shrinkage) | medium |
| BLFX-6 | Hierarchical Platt with per-source intercept offsets | medium |

### Phase 3 — Inputs and priors

| Ticket | Title | Confidence |
|--------|-------|-----------|
| BLFX-7 | Source-specific empirical priors π_q | high |
| BLFX-8 | Crowd signal injection for market questions | low |

### Phase 4 — Backtest integrity

| Ticket | Title | Confidence |
|--------|-------|-----------|
| BLFX-9 | 4-layer date-leakage defense | medium |
| BLFX-15 | Source-specific data tools (yfinance, FRED, Wikipedia-as-of-date) | medium |

### Phase 5 — Benchmark infrastructure

| Ticket | Title | Confidence |
|--------|-------|-----------|
| BLFX-S01 | Spike: ForecastBench dataset access | n/a |
| BLFX-S03 | Spike: cost estimate for one full benchmark run | n/a |
| BLFX-S04 | Spike: base LLM choice (Sonnet vs Gemini-3.1-Pro) | n/a |
| BLFX-10 | ForecastBench dataset ingestion | medium |
| BLFX-11 | Scoring + stats framework (Brier Index + bootstrap CI) | high |
| BLFX-12 | Backtest runner | medium |

### Phase 6 — Verification

| Ticket | Title | Confidence |
|--------|-------|-----------|
| BLFX-13 | Faithful benchmark + ablation sweep | low |

## Risks → Spikes

| Risk | Spike | Fallback if spike fails |
|------|-------|------------------------|
| ForecastBench dataset isn't openly accessible at the question + outcome level we need | BLFX-S01 | Fall back to public Metaculus API + manual question curation; benchmark on a smaller custom set |
| Action format incompatible with Claude Code's tool-use loop | BLFX-S02 | Use a custom prompt-engineered JSON action protocol with retry-on-parse-error; accept higher loop overhead |
| Cost per benchmark run prohibitive (>$500) | BLFX-S03 | Run on smaller question subset (50 instead of 400); use Sonnet instead of Opus to cut model cost; defer benchmark to specific milestones |
| Sonnet substantially worse than Gemini-3.1-Pro on this task | BLFX-S04 | Document the substitution; report numbers as "BLF on Sonnet" not as direct paper-comparison; explore Anthropic API + Opus for parity |

## Out of scope

- **Brave Search specifically.** The paper uses Brave; we use whatever Claude Code's WebSearch backs onto. Engine-level comparability is not the point.
- **Multi-class forecasting.** Stay binary. Murphy 2026 is binary; multi-class is future work.
- ~~**Source-specific data tools beyond what Claude Code provides.**~~ Reinstated as in-scope per BLFX-15 (filed 2026-04-26). The paper's `fetch_ts_yfinance`, `fetch_ts_fred`, `fetch_wikipedia_section` are needed for BLFX-7 (empirical priors) and BLFX-9 (date-leakage defense via Wikipedia revision-as-of-date) to be correct.
- **Reproducing the human-superforecaster baseline.** The paper compares to ABI=70.9 from ForecastBench leaderboard. We just need to hit BLF's number, not the human number.
- **AIBQ2 dataset.** Paper used it for initial development (113 questions). We focus on ForecastBench; AIBQ2 can be added later if useful.

## Calibration

Append a one-line note when BLFX-13 reaches `Complete`: which low-confidence tickets needed contract revision after their spike, what we learned about cost-per-run, what our actual BI was vs paper's reported BI.
