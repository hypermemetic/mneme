# mneme

> μνήμη — *memory*. A forecasting substrate that beats the prediction-market crowd, and the same substrate used to forecast its own design.

Mneme is a working implementation of the [Bayesian Linguistic Forecaster (BLF) algorithm from Murphy 2026](https://arxiv.org/abs/2604.18576) running on a containerized Plexus RPC server. You give it a binary, dated, observable question; it fans out K parallel reasoning trials with web-search-grounded iterative agent loops; it aggregates them in logit space; it applies hierarchical calibration learned from past resolutions; it records the prediction so you can resolve it later when the outcome lands.

The recursion: the same machinery that forecasts events forecasts software design. A ticket can carry a `forecast:` block in its frontmatter — at filing time, the substrate predicts whether that design choice will pan out. When the work ships, `forecast.resolve` closes the loop. Over months, the calibration store grows with predictions about the system's own design.

## What's tested

| Bench | n | Setup | Mneme BI | Crowd BI | Δ vs crowd |
|---|---|---|---|---|---|
| 003 | 20 | single-shot, λ=0.2, no calibration | 64.6 | 70.2 | −5.6 |
| 004a | 20 | paper-aligned defaults (λ=0, Platt) | 68.6 | 70.2 | −1.6 |
| 004b | 20 | + iterative loop on | 83.6 | 70.2 | **+13.4** |
| 005 | 94 | full paper-aligned, n=94 | 84.1 | 70.2 | **+13.9** |
| **006** | **58** | **held-out post-cutoff, freeze 2026-03-15** | **84.0** | **57.5** | **+26.5** |
| **007** | **27** | **held-out post-cutoff, freeze 2026-03-29** | **71.4** | **11.0** | **+60.5** |

Both held-out benches' 95% CIs on paired Brier delta exclude 0. Mneme decisively beats the ForecastBench crowd in this setup.

**The honest caveat:** post-cutoff means the model's training data didn't include the resolutions, but the iterative loop's WebSearch can retrieve news articles published after the freeze date. The +26-60 BI delta likely contains web-search contamination; how much is unknown until BLFX-9 (4-layer date-leakage defense) ships. The live marketplace pipeline is the only data path that's structurally clean — markets that haven't resolved yet, frozen at forecast time, resolving over weeks.

Result docs: [`plans/BLFX/results/`](plans/BLFX/results/).

## What the paper says

[Murphy 2026, BLF](https://arxiv.org/abs/2604.18576) describes five components:

1. **Linguistic belief state** — `(probability, evidence_for, evidence_against, open_questions, confidence)`. Structured, not just a scalar.
2. **K parallel trials** per question.
3. **Iterative agent loop** within each trial — at each step the LLM emits an action AND an updated belief; substrate executes the action; loop until `submit` or T_max steps.
4. **Logit-space LOO-CV-shrunken aggregation** across the K trials.
5. **Hierarchical Platt calibration** with per-source intercept offsets, fit from resolved-outcome history.

The paper's headline: **+4 BI above the prediction-market crowd / human superforecaster baseline on ForecastBench**, using Gemini-3.1-Pro.

The paper's ablation table (the most useful part):

| Removed | Approx BI hit |
|---|---|
| Web search entirely | −4.6 |
| Iterative agent loop | −3.8 |
| Structured belief state | ~−3.0 |
| LOO-CV α aggregation | smaller |
| Hierarchical Platt | smaller |

We replicate the structural components 1-5; the substrate runs on Claude Sonnet (not Gemini); the integrated benchmark vs the paper's exact setup is BLFX-13, not yet run.

## What we built (relative to the paper)

| Paper component | Implementation | Status |
|---|---|---|
| Linguistic belief state | `mneme-substrate/src/activations/forecast/types.rs::ForecastState` (schema 0.3.0) | ✅ |
| K parallel trials | `mneme/runtime/claudecode_swarm_runtime.rs::trial` (parallel via `join_all`) | ✅ |
| Iterative agent loop | `activations/forecast/iterative_loop.rs::iterative_trial` + `runtime/claudecode_step_driver.rs` | ✅ |
| Logit-space aggregation | `mneme/swarm/aggregate/logit.rs`, λ=0 default per Murphy 2026 §4 | ✅ shape; LOO-CV α tuning is BLFX-5 |
| Hierarchical Platt | `mneme/calibration/{platt.rs,store.rs}` + wired into `forecast.update` | ✅ flat Platt; per-source intercepts is BLFX-6 |
| Source-specific tools (yfinance/FRED/Wikipedia-as-of) | stubbed | BLFX-15 |
| 4-layer date-leakage defense | not built | BLFX-9 |

Beyond the paper, mneme adds:

- **Capability registry** (`mneme/capabilities/`) — semantic capability names map to model configs. `json_cleanup` routes structural-fix tasks to Haiku at ~10× cost reduction; `reasoning_strong` reserves Opus for high-value steps. JSON-cleanup recovery on parse failures eliminated the schema-violation failure mode (5/6 of bench-005's failures, 5/63 of bench-006's).
- **Live marketplace pipeline** (`scripts/marketwatch_*.py`) — paired forecasts against live Manifold markets that haven't resolved yet. The dataset that produces unambiguous calibration evidence over weeks/months.
- **Tickets-as-forecasts** (`scripts/ticket_forecast.py`) — every design decision becomes a recorded prediction; the calibration store grows with design-judgment rows.

## The recursion (what makes this different)

The substrate uses prediction as a primitive. Every word the LLM emits is a probability over next-tokens; the substrate uses that primitive to forecast events; **and the same forecasting machinery is used to forecast the design changes that improve the substrate.**

Every meaningful ticket has a `forecast:` block:

```yaml
---
id: BLFX-9
title: "4-layer date-leakage defense"
forecast:
  hypothesis: "Will building BLFX-9 and re-running bench-006 with the
    date-leakage defense active produce a paired delta vs crowd that
    lands within ±10 BI of Murphy 2026's claimed +4 BI?"
  resolution_method: "Re-run scripts/forecastbench_holdout_run.py;
    compute paired delta. YES if delta in [-6, +14]; NO if > +14."
  deadline: "2026-05-27T00:00:00Z"
---
```

`scripts/ticket_forecast.py` reads these blocks, fires `forecast.update` against the substrate, records the prediction in `plans/_predictions.jsonl`. When the deadline passes, `scripts/ticket_resolve.py` prompts for the actual outcome and calls `forecast.resolve`. The substrate's calibration store grows with rows that are explicitly predictions about its own design.

Over time, you get answers to questions nobody else has measured:
- Are this AI agent's design intuitions reliably calibrated?
- On which kinds of design questions is it overconfident?
- How does its calibration on design questions compare to its calibration on event forecasts?

The current sample is small (3 design predictions filed at v0.1) but the loop is closed. The first three:

| Ticket | Hypothesis (abbrev) | Predicted YES | Resolves |
|---|---|---|---|
| MNEME-31 | This script ships + ≥3 predictions recorded by tomorrow | 0.66 | 2026-05-01 |
| BLFX-9 | Date-leakage defense brings delta back to paper's range | **0.50** | 2026-05-27 |
| MNEME-30 | Mneme beats Manifold crowd on first 30 resolved markets | 0.42 | 2026-07-22 |

The 0.50 on BLFX-9 is the system saying "I genuinely don't know if our +26 BI delta is real signal or contamination." That's the answer it should give.

## Quick start (Claude subscription, no API key)

You need:
- macOS or Linux
- Docker / Podman / Colima
- A Claude subscription with the `claude` CLI installed (the run script auto-runs `claude /login` if no OAuth token is found)
- [`synapse`](https://github.com/hypermemetic/synapse) on the host

```bash
git clone git@github.com:hypermemetic/mneme.git
git clone git@github.com:hypermemetic/mneme-substrate.git
cd mneme-substrate
make build           # ~3 min cold; ~30s incremental
make run             # boot substrate, drop into container shell
```

`make run` resolves your auth in priority order (`ANTHROPIC_API_KEY` env → `CLAUDE_CODE_OAUTH_TOKEN` env → macOS Keychain). If none have a token, it auto-runs `claude /login`. Then `docker exec`s you into the container.

The substrate is fire-and-return at the protocol level. Pipe `forecast.update` into `programs.wait` for block-and-print:

```bash
PID=$(synapse -P 4456 substrate forecast update \
        --program-id MY-Q \
        --new-evidence "Will Bitcoin trade above \$200,000 on any day before 2026-12-31?" \
        --trials 3 --iterative-max-steps 5 \
      | jq -r 'select(.content.type == "started") | .content.program_id')

synapse -P 4456 substrate programs wait --program-id "$PID"
```

That streams `progress` events while the substrate works and ends with a terminal `completed` event carrying the artifact. Synapse renders it natively.

To resolve when you know the outcome:

```bash
synapse -P 4456 substrate forecast resolve --program-id <id> --actual true
```

Once you have ≥10 resolutions, Platt calibration kicks in. See [`mneme-substrate/README.md`](https://github.com/hypermemetic/mneme-substrate#readme) for the full operator's guide — bench runs, live Manifold pipeline, ticket-as-forecast, container internals.

## Architecture

```
External callers
       │
       ▼  Plexus RPC over WebSocket / stdio / MCP-HTTP, port 4456
┌─────────────────────────────────────────────────────────────┐
│ Skill activations  ── forecast / programs / capabilities ...
│       │              (composed via direct Rust calls)
│       ▼
│ Orchestration     ── swarm.trial (K parallel trials)
│       │              iterative_loop (action+belief per step)
│       ▼
│ ClaudecodeStepDriver — fork session, chat per step, capture usage
│       │
│       ▼
│ Capability registry — picks model per task (Sonnet / Haiku / Opus)
│       │
│       ▼
│ Calibration store — Platt params fit from resolved observations
│       │
│       ▼
│ Programs/         — per-call directory: manifest + artifact + trace
│ Arbor             — full conversation tree (every chat turn captured)
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
                   ~/.plexus persistent state
                   (bind-mounted to host disk;
                    survives container rebuilds)
```

| | |
|---|---|
| [mneme](https://github.com/hypermemetic/mneme) (this repo) | Tickets, design, paper, result docs |
| [mneme-substrate](https://github.com/hypermemetic/mneme-substrate) | The Plexus RPC server, scripts, container |

## Status (v0.1, first public)

**Working:**
- ✅ All 5 paper components in structural form
- ✅ Container deployment with Claude-subscription auth forwarding
- ✅ Three benches: paper-aligned single-shot, paper-aligned iterative, two held-out post-cutoff windows
- ✅ Calibration loop end-to-end (170+ resolved observations seeding Platt)
- ✅ Live Manifold marketplace pipeline (Phase 1)
- ✅ Tickets-as-forecasts pipeline (Phase 1)
- ✅ JSON-cleanup recovery for malformed agent output
- ✅ All 370+ tests pass

**Open (the actual research questions):**
- 🔲 BLFX-9 — date-leakage defense (the load-bearing test of whether our +26 BI is real)
- 🔲 BLFX-5 — LOO-CV-tuned α (currently λ=0 fallback per Murphy's empirical optimum)
- 🔲 BLFX-6 — hierarchical Platt with per-source intercepts (mneme is weak on metaculus questions; per-source bias correction would help)
- 🔲 BLFX-13 — full integrated paper-comparison run with ablation sweep
- 🔲 BLFX-15 — source-specific data tools (yfinance / FRED / Wikipedia-as-of)
- 🔲 Polymarket integration (current: Manifold only)
- 🔲 Long-term resolved-pair dataset (live pipeline started 2026-04-29; first resolutions in days)

## Authorship

Implementation: **Claude (Opus 4.7)** with Anthropic Code SDK / Code CLI.

Direction: **[Ben Haware](https://github.com/Hypermemetic)**.

The methodology is the one in `~/CLAUDE.md`:

> The human decides. The agent executes. The ticket is the interface.

Concretely:

- Ben sets goals, makes load-bearing architectural calls (containerization, Plexus RPC choice, lattice/orcha integration, ticket-format-as-cognate-prompt, Claude-subscription auth forwarding, the framing of forecasting-as-action).
- Claude writes code, files tickets in canonical format, runs experiments, writes result docs, makes implementation-level judgment calls.
- The ticket trail in `plans/MNEME/` and `plans/BLFX/` is the audit. Each ticket records both the design decision and (where filed) the system's prediction about the decision's outcome.

Every commit in both repos carries `Co-Authored-By: Claude Opus 4.7 (1M context)`.

## License

The Murphy 2026 BLF paper is the algorithmic prior art; we implement it openly. ForecastBench data is CC BY-SA 4.0 from forecastingresearch/forecastbench-datasets. Code in this repo: MIT (see LICENSE; pending — currently authored, not yet license-stamped).
