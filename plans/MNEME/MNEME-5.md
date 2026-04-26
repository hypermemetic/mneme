---
id: MNEME-5
title: "`swarm.aggregate` — logit shrinkage + concat-evidence"
status: Pending
type: implementation
blocked_by: [MNEME-4]
unlocks: [MNEME-6]
confidence: high
---

## Problem

After `swarm.trial` returns N typed responses, the caller needs to combine them into one. For BLF this is logit-space shrinkage toward a prior; for security-review it's max-severity with deduplication; for any prose summary it's concatenation with attribution. Each skill should not reimplement these aggregations.

## Context

- BLF aggregation: convert each probability `p_i` to logit `l_i = log(p_i / (1-p_i))`, take weighted mean shrunk toward prior logit `l_0` with shrinkage factor `λ`, convert back. Mechanical math; no LLM needed.
- Skills also need non-numerical aggregations: `concat_evidence` (gather per-trial summaries with trial index attribution), `majority_enum` (vote on a discrete value), `max_severity` (for security findings).

## Evidence

The aggregation rules are mechanical and deterministic — keeping them out of the LLM path means (a) the math is testable, (b) results are reproducible, (c) they cost nothing per call. The whole point of BLF's logit-shrinkage is that calibration is a *math* operation on numbers an LLM produced, not a second LLM judgment call.

Confidence is `high` because this is pure-Rust math against well-defined inputs, no substrate or model involvement, and the test cases are obvious (compare against scipy / by-hand calculations).

## Required behavior

`swarm.aggregate` method on the `swarm` activation:

```
swarm.aggregate(
    trials: Vec<Value>,            // typically the .response fields from swarm.trial
    rule: AggregationRule,
) -> Stream<AggregateEvent>
```

`AggregationRule` is a tagged enum with these variants for MVP:

| Rule | Inputs (per trial) | Output | Notes |
|------|-------------------|--------|-------|
| `LogitShrinkage { field, prior, lambda }` | object with `field: number in (0,1)` | `{ aggregated: number, raw_mean: number, n: u8 }` | BLF shrinkage. `lambda=0` = mean of trials. `lambda=1` = prior. |
| `ConcatEvidence { field, separator }` | object with `field: string` | `{ aggregated: string }` | Joins per-trial strings with `separator`, prefixed `[trial i]:` |
| `MajorityEnum { field }` | object with `field: string` | `{ aggregated: string, votes: object }` | Most-common value; ties broken by lexicographic order |
| `MaxSeverity { field, ladder }` | object with `field: string` | `{ aggregated: string, count: integer }` | Max along the ladder (e.g., `["Low","Medium","High","Critical"]`) |

Events:

| Variant | Fields |
|---------|--------|
| `Aggregated` | `result: Value` (matches the rule's output schema) |
| `Err` | `message: String` (e.g., trial missing the named field) |

Behavior:

| Scenario | Outcome |
|----------|---------|
| `trials.len() == 0` | `Err("no trials to aggregate")` |
| `trials.len() == 1` | Returns the single trial's value passed through (no shrinkage applied, since there's nothing to shrink toward). For `LogitShrinkage`, the `prior` is still applied via `lambda`. |
| Trial missing the named `field` | `Err` with the trial index that was malformed |
| Numerical field out of `(0,1)` for `LogitShrinkage` | `Err`; clamp is intentional caller responsibility |

## Risks

| Risk | Mitigation |
|------|-----------|
| `lambda` is not the right shrinkage parameterization for downstream skills | Expose both `lambda` (interpolation) and a `prior_n` (effective sample size) helper in docs; pick the one the forecast skill actually needs |
| Aggregations grow without bound as new skills land | Keep MVP at four; new rules require their own ticket with test cases |
| Numerical edge cases (p=0 or p=1) cause logit overflow | Clamp inputs to `[1e-6, 1 - 1e-6]` before logit; document the clamp |

## What must NOT change

- The `swarm` activation namespace or its `swarm.trial` method.
- The TrialEvent shape from MNEME-4 (this method consumes `Vec<Value>`, agnostic to where they came from).

## Acceptance criteria

1. `swarm.aggregate` registered as a method on the existing `swarm` activation.
2. Unit tests in `mneme/src/swarm/aggregate_test.rs`:
   - `logit_shrinkage_no_prior_pull`: trials = [0.5, 0.5, 0.5], prior=0.3, lambda=0 → 0.5.
   - `logit_shrinkage_full_pull`: trials = [0.5, 0.5, 0.5], prior=0.3, lambda=1 → 0.3.
   - `logit_shrinkage_partial`: trials = [0.8, 0.7, 0.9], prior=0.5, lambda=0.5 → matches hand-computed value within 1e-6.
   - `concat_evidence`: 3 trials with `summary` strings → output joined with `[trial 1]: ...\n\n[trial 2]: ...\n\n[trial 3]: ...`.
   - `majority_enum_clear`: trials with field values `["a","a","b"]` → `"a"`.
   - `majority_enum_tie`: trials with `["a","b"]` → lexicographic loser; document which.
   - `max_severity`: trials with `["Low","High","Medium"]` along ladder → `"High"`.
   - `empty_trials_errors`: `trials.len() == 0` → `Err`.
   - `missing_field_errors`: one trial without `field` → `Err` with that trial's index.
3. `cargo build` and `cargo test` pass green for the `mneme` crate.

## Completion

Implementor runs `cargo test --lib swarm::aggregate`, all pass; status flips to `Complete` in the same commit as the code.

## Implementation status (2026-04-26 autonomous session)

**Done.** All four aggregation rules implemented and tested in `mneme-substrate/src/mneme/swarm/aggregate/`: `logit.rs` (BLF logit-shrinkage with input clamping), `concat.rs` (per-trial summary join), `majority.rs` (lex-tie-break majority), `severity.rs` (max along ladder). Dispatcher in `aggregate/mod.rs`. The math is exercised end-to-end by `pipeline_e2e_program_trial_aggregate_calibrate_artifact`. The aggregation runs as pure Rust (no Plexus method yet); when MNEME-4's swarm activation is wired into the substrate, `swarm.aggregate` will become the public Plexus method that wraps `aggregate::aggregate(...)`.
