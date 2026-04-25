# mneme

> μνήμη — *memory*; the substrate that carries belief forward across agents.

Mneme is a Plexus-native harness that turns the hypermemetic skill suite (`ticketing`, `planning`, `security-review`, `strong-typing`, `forecast`) into platform services. Each skill is a typed Plexus activation. Each invocation spawns one or more Claude Code subagents via the existing `claudecode` activation in `plexus-substrate`, records the full execution as a directory artifact, and returns a structured response that downstream callers can condition on.

The harness is the runtime that makes the BLF principle — *every artifact carries forward the evidence that justifies it* — operational at the protocol layer rather than at the documentation layer.

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

*The competitive frontier mneme inherits. The harness's job is to make these techniques composable and re-usable across skills, not just forecasting.*

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ Layer 3: Harness binary  (`mneme`)                          │
├─────────────────────────────────────────────────────────────┤
│ Layer 2: Skill activations                                  │
│   forecast / ticketing / planning / security_review / ...   │
├─────────────────────────────────────────────────────────────┤
│ Layer 1: Orchestration primitives  (`swarm`, `respond`)     │
│   trial / aggregate / sequential / race                     │
├─────────────────────────────────────────────────────────────┤
│ Layer 0: claudecode activation  (in plexus-substrate)       │
│   sessions, fork, loopback, persistence                     │
└─────────────────────────────────────────────────────────────┘
```

The four layers are independently testable and replaceable. Layer 0 is upstream (lives in `plexus-substrate`); the rest is what this repo builds.

## Status

Pre-implementation. Tickets are written but no code exists yet. See [`plans/MNEME/MNEME-1.md`](plans/MNEME/MNEME-1.md) for the epic and the phase breakdown.

| Phase | Goal | Status |
|-------|------|--------|
| 1 | One skill (`forecast`) end-to-end through the full stack with recording | Tickets pending |
| 2 | Port the rest of the skill suite (`ticketing`, `planning`, `security_review`, `strong_typing`) | Tickets pending |
| 3 | Sequential / race primitives + calibration loop closure | Tickets pending |

All tickets currently sit at `status: Pending`. Per the methodology, tickets need explicit human approval before flipping to `Ready` for implementation.

## Layout

```
mneme/
  README.md
  plans/
    MNEME/
      MNEME-1.md         epic overview
      MNEME-S01..S03.md  spikes
      MNEME-2..15.md     implementation tickets
  docs/
    blf-paper.pdf        local copy of the inspiration paper
    figures/             figures extracted from the paper
  programs/              (created at first run; one subdir per invocation)
```

## Methodology pointers

The skill suite mneme orchestrates lives at `~/dev/controlflow/hypermemetic/skills/skills/`. The methodology that informs ticket structure (the "two-stranger test", the `## Evidence` section, the `confidence:` frontmatter, the spikes-as-evidence-sources principle) lives in those skills' `SKILL.md` files and in `~/CLAUDE.md`.

This repo's tickets exemplify the discipline they describe — every implementation ticket carries a real `## Evidence` section, every spike has a binary pass condition plus structured evidence to record, and every confidence value is set deliberately so post-MVP calibration has data to learn from.
