---
id: MNEME-6
title: "`forecast` skill activation (end-to-end)"
status: Pending
type: implementation
blocked_by: [MNEME-3, MNEME-4, MNEME-5]
unlocks: [MNEME-7]
confidence: low
---

## Problem

The four-layer architecture is unverified until one skill rides the full stack — Layer 0 (`claudecode`) → Layer 1 (`swarm.trial` + `swarm.aggregate` + `respond`) → Layer 2 (a typed skill activation) — and produces a structured result a user can act on. `forecast` is the smallest such skill and it directly exercises BLF, which is the harness's load-bearing inspiration.

## Context

- BLF (the paper at `docs/blf-paper.pdf`) defines a forecast as `(probability, evidence_summary)` where the summary is the sufficient statistic for the next update.
- The `forecast` skill described in `~/dev/controlflow/hypermemetic/skills/skills/` (designed but never implemented) uses N parallel trials, logit-shrinkage aggregation, and an explicit resolvability gate.
- The skill's `update` method accepts a prior `(probability, summary)` and new evidence, runs trials conditioned on the prior, aggregates, returns the new state.

## Evidence

`forecast` is the right validation skill — not `ticketing` or `planning` — because:

1. It exercises every layer-1 primitive in one workflow (`respond` for typed output, `swarm.trial` for fan-out, `swarm.aggregate` with both `LogitShrinkage` and `ConcatEvidence`).
2. The output shape is small and easy to inspect (`{probability, summary}`), so test assertions are mechanical.
3. The BLF semantics are well-defined; we don't have to invent the skill's contract on top of inventing the harness.

Confidence is `low` because this is the first skill end-to-end. Two failure modes are likely:
- Layer 1's contract is subtly wrong for what a real skill needs, requiring revisions to MNEME-3..5.
- The `respond` tool round-trip is too slow at N=5 to be usable, forcing N=1 default.

If MNEME-S02 reports correlated forks, this ticket's confidence drops further and the trial count defaults to 1 with prominent `confidence: single-pass` tagging in the output.

## Required behavior

`forecast` activation, namespace `forecast`, version `0.1.0`. Methods:

```
forecast.create(
    question: String,           // binary, dated, observable
    resolution_criterion: String,
    deadline: String,           // ISO 8601 date
    trials: Option<u8>,         // default 3
) -> Stream<CreateEvent>

forecast.update(
    program_id: String,         // returned by create or prior update
    new_evidence: String,       // user-supplied, may be ""
    trials: Option<u8>,
) -> Stream<UpdateEvent>
```

Events from `forecast.update` (the load-bearing one):

| Variant | Fields |
|---------|--------|
| `Started` | `program_id: String`, `prior: ForecastState` |
| `ResolvabilityFailed` | `reason: String` (e.g., "deadline in past") |
| `TrialProgress` | `trial_index: u8`, `total: u8`, `status: String` |
| `Aggregated` | `aggregated: ForecastState`, `raw_trials: Vec<ForecastState>` |
| `Completed` | `program_id: String`, `state: ForecastState`, `artifact_path: String` |
| `Err` | `stage: String, message: String` |

`ForecastState` shape:

| Field | Type | Notes |
|-------|------|-------|
| `probability` | number `[0, 1]` | The point estimate |
| `summary` | string | Linguistic belief (BLF sufficient statistic) |
| `confidence` | enum | `multi-trial` \| `single-pass` |
| `n_trials` | u8 | Number of trials that contributed |
| `prior_used` | object \| null | The prior this update conditioned on; null on first call |

Internal flow for `update`:

1. Read prior from `programs/<program_id>/belief_state.md` (or `manifest.inputs` if first call).
2. Resolvability gate: if deadline passed, fail with explanation.
3. Compose system prompt = `forecast/SKILL.md` (baked via `include_str!`) + question + criterion + prior summary.
4. Create or fork a parent session with that system prompt.
5. Build prompt = "New evidence: {evidence}. Produce a probability and a one-paragraph evidence summary."
6. Call `swarm.trial` with response schema = `{probability: number, summary: string}`, n=trials, diversify="Reasoning style #%i (analytic / contrarian / base-rate-grounded)".
7. Call `swarm.aggregate` with `LogitShrinkage { field: "probability", prior: prior.probability, lambda: 0.2 }` AND `ConcatEvidence { field: "summary", separator: "\n\n" }` (two passes).
8. Write new state to `belief_state.md`; append to `updates/<seq>_<iso-date>.md`.
9. Emit `Completed` with the artifact path.

## Risks

| Risk | Mitigation |
|------|-----------|
| `lambda=0.2` is wrong for first-update vs. tenth-update | Phase 1 hardcodes 0.2; phase 3 (MNEME-15 calibration loop) replaces with a learned schedule |
| Resolvability gate is too strict (refuses real forecasts) | Gate refuses only deadline-in-past; unobservable criteria flagged with a warning event but proceed |
| Skill's `update.md` prompt is brittle in surprising ways | Spec a prompt smoke-test: 5 known questions with known answers; CI runs them and asserts probability is in expected band |

## What must NOT change

- The `swarm` activation API. If forecast needs something swarm doesn't provide, file a separate ticket.
- The program directory layout from MNEME-2.

## Acceptance criteria

1. `forecast` activation registered.
2. `forecast/SKILL.md` baked via `include_str!` matches `~/dev/controlflow/hypermemetic/skills/skills/forecast/SKILL.md` (or whatever forecast skill exists in the source repo at the time of implementation).
3. Integration test `tests/forecast_e2e.rs`: creates a forecast for "Will Bitcoin close above $100,000 USD on 2026-12-31?" with deadline `2026-12-31`, calls `update` with empty evidence, asserts the response has a probability in `[0, 1]` and a non-empty summary, asserts a `programs/<id>/` directory exists with the right files (per MNEME-2).
4. Integration test `tests/forecast_resolvability.rs`: question with deadline `2024-01-01` returns `ResolvabilityFailed`.
5. Integration test `tests/forecast_update_chain.rs`: create + 3 sequential updates; asserts the prior in update N matches the aggregated state from update N-1 (the BLF carry-forward property).
6. `cargo build` and `cargo test` pass green for the `mneme` crate.

## Completion

Implementor runs `cargo test --test forecast_e2e forecast_resolvability forecast_update_chain`, all pass; status flips to `Complete` in the same commit as the code. Calibration history file `programs/_calibration/history.jsonl` initialized empty in the same commit.
