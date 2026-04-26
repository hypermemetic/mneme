# mneme

> μνήμη — *memory*; the substrate that carries belief forward across agents.

Mneme is a Plexus RPC server (a fork of `plexus-substrate`) that turns the hypermemetic skill suite (`ticketing`, `planning`, `security-review`, `strong-typing`, `forecast`) into platform services. Each skill is a typed Plexus activation. Each invocation spawns one or more Claude Code subagents via the `claudecode` activation, records the full execution as a directory artifact, and returns a structured response that downstream callers can condition on.

Mneme is the runtime that makes the BLF principle — *every artifact carries forward the evidence that justifies it* — operational at the protocol layer rather than at the documentation layer.

## What it gives you

```
$ mneme run forecast.update --program-id Q-001 --new-evidence "BTC at $95k"
program_id: 8c1f3e0a-...
artifact:   programs/8c1f3e0a-.../artifact.json
{
  "probability": 0.34,
  "summary": "Three trials disagreed on whether BTC's recent rally is durable...",
  "confidence": "multi-trial",
  "n_trials": 3,
  "prior_used": { "probability": 0.41, "summary": "..." }
}
```

The `programs/<id>/` directory IS the run: manifest, artifact, JSONL trace, captured Claude sessions, child program subdirectories. Replay, audit, and calibration all read this directory.

Synapse works against the same backend with no extra wiring:

```
$ synapse -P 4444 forecast update --program-id Q-001 --new-evidence "BTC at $95k"
```

## This repo's role

This repository (`mneme/`) is the **planning and design** repo: tickets, architecture decisions, the BLF paper figures, the issues log. The implementation lives in a sibling repo (`mneme-substrate/`) that was forked from `plexus-substrate` and evolves toward the architecture this repo's tickets describe.

| Repo | Role |
|------|------|
| `mneme/` (this repo) | Planning artifacts: 18 tickets, ISSUES log, BLF paper + figures, architecture |
| `mneme-substrate/` (sibling) | Implementation: forked Plexus RPC server with the `claudecode`, `swarm`, and skill activations |

Keeping them split lets the planning artifacts evolve at a different pace from the code, and lets the implementation get aggressive with refactors without churning the design history.

## Inspiration

Mneme operationalizes the framework from:

> **Agentic Forecasting using Sequential Bayesian Updating of Linguistic Beliefs**
> Kevin Murphy, arXiv [2604.18576](https://arxiv.org/abs/2604.18576), April 2026
> [(local copy)](docs/blf-paper.pdf)

The paper introduces a binary-forecasting agent (BLF — Bayesian Linguistic Forecaster) whose belief state isn't just a probability but a `(probability, evidence_summary)` pair that survives between updates. Multi-trial aggregation in logit space and Platt-scaling calibration produce a system that beats GPT-5, Grok 4.20, and Foresight-32B on ForecastBench backtests.

The mneme generalization: every artifact in an agent pipeline — tickets, plans, security findings, code reviews — should carry forward both its value AND the evidence justifying it. Multi-trial aggregation and post-hoc calibration are the same primitives across all of them.

### Belief evolution under multi-trial

![Five trials' belief probabilities over agent steps for a binary question, with the aggregated mean.](docs/figures/fig01-belief-evolution.png)

*Each trial is an independent reasoning path; aggregation in logit space produces the mean line. The same primitive in mneme is `swarm.trial(n=5) → swarm.aggregate(LogitShrinkage)`.*

### Why multiple trials beat single trials

![Four-panel ablation: as the number of forecasts averaged increases, Metaculus Baseline Score, Brier Score, and Brier Index all improve; LOO-shrinkage (blue) consistently beats plain mean (gray).](docs/figures/fig02-trial-aggregation.png)

*The ntrials axis is the BLF multi-trial parameter — `swarm.trial`'s `n`. The LOO-shrinkage curve is what `swarm.aggregate` with `LogitShrinkage` reproduces.*

### Where BLF sits on the leaderboard

![Forecasting leaderboard: Superforecaster median forecast, Cassi ensemble, Gemini-3-Pro-Preview, Grok 4.20, GPT-5, Foresight-32B — with Dataset, Market, and Overall scores plus 95% CIs.](docs/figures/fig03-leaderboard.png)

*The competitive frontier mneme inherits. Mneme's job is to make these techniques composable and re-usable across skills, not just forecasting.*

## Architecture

mneme-substrate is **one Rust binary**. Plexus RPC is the boundary protocol exposed to outside callers (synapse, MCP, WS). Inside the binary, modules compose via normal Rust function calls — Plexus is not the internal calling convention.

```
┌──────────────────────────────────────────────────────────────┐
│ External callers (synapse, MCP, WS clients)                  │
└────────────────────────┬─────────────────────────────────────┘
                         │ Plexus RPC over WS / stdio / HTTP
┌────────────────────────▼─────────────────────────────────────┐
│ mneme-substrate (single binary)                              │
│                                                              │
│  Skill activations  ── forecast / ticketing / planning /     │
│       │                 security_review / strong_typing      │
│       │  (Rust calls, not RPC)                               │
│       ▼                                                      │
│  Orchestration      ── swarm.trial / aggregate /             │
│       │                 sequential / race                    │
│       │  (Rust calls)                                        │
│       ▼                                                      │
│  claudecode         ── sessions, fork, loopback, persistence │
│       ▲                                                      │
│       │ MCP loopback (real boundary — Claude inside session  │
│       │              calls back via per-program tool reg)    │
│  Substrate runtime  ── program lifecycle (wraps dispatch)    │
│                        per-program tool registry             │
│                        session attribution                   │
│                        calibration store                     │
│                        schema enforcement on returns         │
└──────────────────────────────────────────────────────────────┘
```

Key principles:

- **One dispatch path = one observability surface.** Skill A calling skill B in-process goes through Rust, not RPC. The substrate's program-lifecycle middleware wraps the *external* dispatch boundary so every top-level call becomes a recorded program; internal composition is recorded by the substrate's awareness of swarm calls.
- **Programs are substrate state, not Plexus state.** Plexus method shapes are unchanged from upstream; the substrate just brackets each top-level call with program open/close.
- **Loopback is the only place we cross a serialization boundary internally.** When Claude *inside* a session calls back via MCP, that's a real principal change and the per-program `respond` tool registry handles it.

## Status

In-flight. The plans are written; implementation is starting.

| Phase | Goal | Status |
|-------|------|--------|
| 0 | Fork plexus-substrate → mneme-substrate (sibling repo); structural changes only, no behavior change | In progress |
| 1 | Substrate-side modules: program lifecycle, swarm primitives, aggregation math, respond protocol, calibration | Tickets pending |
| 2 | Forecast skill end-to-end (the validation skill) | Ticket pending |
| 3 | Port `ticketing`, `planning`, `security_review`, `strong_typing` as activations | Tickets pending |
| 4 | Sequential / race / calibration loop closure | Tickets pending (deferred until usage justifies) |

All tickets currently sit at `status: Pending`. Per the methodology, tickets need explicit human approval before flipping to `Ready` for implementation. The 18 tickets are in [`plans/MNEME/`](plans/MNEME/); the epic overview is [`MNEME-1.md`](plans/MNEME/MNEME-1.md).

## Issues log

[`ISSUES.md`](ISSUES.md) is the append-only journal of issues discovered during work on mneme and adjacent skills. No issue too small to log. Discipline: every check that surfaces something gets an entry, including warnings that would normally be waved through. A logged-and-deferred issue is fine; an unlogged one rots in conversation history.

## Layout

```
mneme/
  README.md
  ISSUES.md            append-only issues journal
  plans/MNEME/
    MNEME-1.md         epic overview (architecture + DAG + phases)
    MNEME-S01..S03.md  spikes (3)
    MNEME-2..15.md     implementation tickets (14)
  docs/
    blf-paper.pdf      local copy of the BLF paper
    figures/           three curated figures (belief evolution, trial agg, leaderboard)
```

## Methodology pointers

The skill suite mneme orchestrates lives at `~/dev/controlflow/hypermemetic/skills/skills/`. The methodology that informs ticket structure (the "two-stranger test", the `## Evidence` section, the `confidence:` frontmatter, the spikes-as-evidence-sources principle, the bilateral `presence` working posture) lives in those skills' `SKILL.md` files and in `~/CLAUDE.md`.

This repo's tickets exemplify the discipline they describe — every implementation ticket carries a real `## Evidence` section, every spike has a binary pass condition plus structured evidence to record, and every confidence value is set deliberately so post-MVP calibration has data to learn from.
