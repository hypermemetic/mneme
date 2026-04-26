---
id: MNEME-1
title: "Mneme — skill orchestration on Plexus RPC"
status: Epic
type: epic
blocked_by: []
unlocks: []
---

## Goal

A Plexus RPC server that turns the skill suite (`ticketing`, `planning`, `security_review`, `strong_typing`, `forecast`) into platform services. Each skill is a typed activation. Each invocation spawns one or more Claude Code subagents, records the full execution as a directory artifact, and returns a structured response that downstream callers condition on.

Mneme makes the BLF principle — *every artifact carries forward the evidence that justifies it* — operational at the protocol layer rather than at the documentation layer.

## Why

Skills as documentation are unrepeatable, untyped, uncomposable. Mneme changes that:

1. **Skills are functions.** Typed inputs, typed outputs, callable over Plexus RPC.
2. **Invocations are programs.** A directory under `programs/<id>/` is the residue of execution.
3. **Multi-agent fanning is a primitive.** Multi-trial, multi-pass review, parallel spikes — done once, used everywhere.
4. **Skills compose.** Skill A invokes skill B via loopback; the call is recorded as a child program. Composition is recursion at the protocol level.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ mneme CLI                                                   │
│   `mneme run <skill> [args...]` / inspect / programs list   │
├─────────────────────────────────────────────────────────────┤
│ Skill activations                                           │
│   forecast / ticketing / planning / security_review /       │
│   strong_typing                                             │
├─────────────────────────────────────────────────────────────┤
│ Orchestration                                               │
│   swarm: trial / aggregate / sequential / race              │
│   respond: structured-output via per-program loopback tools │
├─────────────────────────────────────────────────────────────┤
│ claudecode  (inherited from substrate)                      │
│   Sessions, fork, loopback, persistence                     │
└─────────────────────────────────────────────────────────────┘
```

## Dependency DAG

```
MNEME-S01 (spike: loopback tool registration)
   │
   └──> MNEME-3 (respond tool + structured-output protocol)
          │
          └──> MNEME-4 (swarm.trial)
                 │
                 ├──> MNEME-5 (swarm.aggregate)
                 │      │
                 │      └──> MNEME-6 (forecast skill — end-to-end validation)
                 │             │
                 │             └──> MNEME-7 (program reification)
                 │                    │
                 │                    └──> MNEME-8 (harness binary)
                 │
                 └──> MNEME-S03 (spike: parallel-fork scaling bound)

MNEME-S02 (spike: fork independence) ──> MNEME-4

MNEME-2 (program directory layout) is a contract document that
unblocks MNEME-7 — can be drafted in parallel with MNEME-3..6.

Phase 2 (skill ports) blocked_by: MNEME-8
   MNEME-9  ticketing as activation
   MNEME-10 planning as activation
   MNEME-11 security_review as activation
   MNEME-12 strong_typing as activation

Phase 3 (extras) blocked_by: usage signal from phase 2
   MNEME-13 swarm.sequential
   MNEME-14 swarm.race
   MNEME-15 calibration loop closure
```

## Phases

### Phase 1 — MVP (one skill end-to-end through the full stack)

Goal: prove the architecture by getting `forecast.update` to round-trip through every layer with recording and structured output.

| Ticket | Title | Confidence |
|--------|-------|-----------|
| MNEME-S01 | Spike: loopback MCP supports tool registration with typed schema | n/a |
| MNEME-S02 | Spike: fork independence under shared system prompt | n/a |
| MNEME-S03 | Spike: parallel-fork scaling bound on substrate | n/a |
| MNEME-2 | Define program directory layout & manifest format | high |
| MNEME-3 | `respond` tool + structured-output protocol | medium |
| MNEME-4 | `swarm.trial` — fan-out via fork | medium |
| MNEME-5 | `swarm.aggregate` — logit shrinkage + concat-evidence | high |
| MNEME-6 | `forecast` skill activation (end-to-end) | low |
| MNEME-7 | Program reification — recording & manifest writer | medium |
| MNEME-8 | `mneme` harness binary | high |

### Phase 2 — Port the rest of the skill suite

Goal: every documented skill becomes a typed activation. Each port is independent of the others.

| Ticket | Title | Confidence |
|--------|-------|-----------|
| MNEME-9 | Port `ticketing` as activation | medium |
| MNEME-10 | Port `planning` as activation | medium |
| MNEME-11 | Port `security_review` as activation | medium |
| MNEME-12 | Port `strong_typing` as activation | high |

### Phase 3 — Extras (when usage justifies)

| Ticket | Title | Confidence |
|--------|-------|-----------|
| MNEME-13 | `swarm.sequential` — sequential update with prior-as-context | medium |
| MNEME-14 | `swarm.race` — first-success-wins for spike chains | medium |
| MNEME-15 | Calibration loop closure (resolve forecasts → update Platt params) | low |

## Risks → Spikes

| Risk | Spike | Fallback if spike fails |
|------|-------|------------------------|
| Loopback MCP can't register a tool with a JSON-schema-constrained input | MNEME-S01 | Drop tool-call structured output; fall back to prompt-engineered JSON parsing with a retry loop |
| Forks from the same parent session produce correlated reasoning paths (BLF assumes independence) | MNEME-S02 | Add prompt-level diversification (different "perspectives" per trial) before falling back to single-trial mode |
| N parallel forks degrade substrate performance below usable threshold | MNEME-S03 | Cap N in `swarm.trial`; fan out across multiple substrate instances |
| Forecast skill isn't enough to validate the whole stack (too narrow) | — | If MNEME-6 ships clean but doesn't exercise multi-trial aggregation meaningfully, validate with `security_review` instead |

## Out of scope

- **Continuous-variable forecasting.** Stay binary. Multi-class is a future epic.
- **Cross-substrate orchestration.** All subagents run in one substrate process. Distributed orchestration is post-MVP.
- **Hot-reload of skill activations.** Skills ship baked-in via `include_str!` of their SKILL.md. A skill registry activation is a phase-3 idea, not phase-1.
- **Full BLF Platt calibration in MVP.** Calibration data accumulates from real resolved forecasts; until ~10 are resolved, calibration is identity-with-shrinkage. MNEME-15 closes the loop later.
- **A web UI.** Programs are filesystem artifacts; the harness is CLI-first. UI is a separate consumer.

## Calibration

When the last phase-1 ticket reaches `Complete`, append a one-line note at the bottom: which `low`-confidence tickets needed contract revision after their spike, which `high`-confidence ones over-ran. Updates the priors for subsequent epic planning.
