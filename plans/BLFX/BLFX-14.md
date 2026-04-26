---
id: BLFX-14
title: "Model-as-first-class-choice at every step + skillsets registry"
status: Pending
type: implementation
blocked_by: []
unlocks: [BLFX-13]
confidence: medium
---

## Problem

Today, the model used for forecasting is hardcoded: `Forecast.update` → `ensure_parent_session` always passes `model: "sonnet"`. Skills can't override; trials inherit; action steps within a trial (when BLFX-4 lands) can't choose a different model. This forces every Claude call to be the same tier, regardless of whether the task needs a frontier model (architecture decisions) or could use a cheap one (summarize_results, format conversion).

The user's directive: *"we need to make sure that the model is a choice at all steps, and we consider which model based on skillsets which we build over time."* Two requirements:

1. **Model is a parameter at every level** — skill activation, trial fan-out, and (when iterative loop lands) per-action step.
2. **Skillsets registry** — accumulated knowledge of which models perform well on which task types. Used to route automatically; updated as we observe outcomes.

## Context

The current model handling:
- `Forecast::update` opens a `parent_session` via `ensure_parent_session` with hardcoded `model: "sonnet"`.
- `claudecode.fork` inherits the parent's model; we can't override per-fork.
- `claudecode.create` accepts a model; it's how we'd opt into a different tier.
- The substrate's `Model` enum is `Opus | Sonnet | Haiku`.

For BLFX (and skills generally), we want:
- A `model` parameter on `forecast.update` (and every skill method) — accepts an optional model name; defaults to a sensible tier-per-skill.
- A `model_per_trial` Vec parameter that lets the caller (or a future meta-controller) pick different models for different trials in the same fan-out (e.g., 4 sonnet + 1 opus for a "tiebreaker" pattern).
- For BLFX-4's iterative loop: per-step model choice (cheap model for `summarize_results`, frontier for the final `submit` step).
- A registry (`skillsets`) that observes `(skill, model, task_type, outcome_quality)` tuples and recommends models based on accumulated evidence.

## Evidence

This generalizes the autonomous-work skill's "match model to task" principle from delegation-time to runtime. The mneme architecture should embody it because:

1. **Cost.** Running every step on Opus is 5x the Sonnet cost. Most steps don't need it.
2. **Latency.** Frontier models are slower; for a pipeline with N steps, picking the right model per step compounds to meaningful latency wins.
3. **Quality.** Some tasks (heavy reasoning, careful instruction-following) benefit from frontier; some (mechanical formatting, summarization) don't and may even be worse on a frontier model that over-elaborates.
4. **Learning.** Without a registry, every developer rediscovers what's good for what. With one, the substrate accumulates the team's collective experience.

The skillsets registry is also load-bearing for the BLFX-13 benchmark: per BLFX-S04, we'll want to compare BLFX-on-Sonnet vs BLFX-on-Opus. The registry stores those results and informs future routing.

## Required behavior

### Layer 1: model parameter at every method

Every skill method accepts an optional `model: Option<String>`:

```rust
forecast.update(program_id, new_evidence, trials?, parent_session?, allowed_tools?, model?)
ticketing.write(epic, problem, ..., model?)
planning.epic(goal, constraints, ..., model?)
```

When `None`, the skill chooses based on:
1. Skillsets registry recommendation (if registry has data), OR
2. The skill's documented default (e.g., forecast defaults to sonnet)

### Layer 2: per-trial model selection

`TrialParams` extended:

```rust
pub struct TrialParams {
    // ...existing fields...
    pub model: Option<String>,                  // single model for all trials
    pub model_per_trial: Option<Vec<String>>,   // per-trial override; len must == n
}
```

`ClaudeCodeSwarmRuntime::trial` interprets:
- If `model_per_trial.is_some()`, use it; each fork gets `claudecode.create` with the matching model.
- Else if `model.is_some()`, use it for all trials.
- Else use the parent session's model (current behavior).

### Layer 3: per-step model selection (lands with BLFX-4)

The iterative agent loop accepts a `step_model_policy`:

```rust
pub enum StepModelPolicy {
    Fixed(String),                              // same model for every step
    PerActionType(HashMap<ActionType, String>), // routing by action
    Adaptive,                                   // skillsets registry decides per-step
}
```

`PerActionType` example:
- `WebSearch` → `sonnet` (research, no deep reasoning yet)
- `SummarizeResults` → `haiku` (mechanical consolidation)
- `LookupUrl` → `sonnet` (page understanding)
- `Submit` → `opus` (final probability requires frontier reasoning)

### Layer 4: skillsets registry

A new SQLite table mirroring the calibration store pattern:

```sql
CREATE TABLE mneme_skillset_observations (
    observation_id TEXT PRIMARY KEY,
    skill_namespace TEXT NOT NULL,              -- forecast, ticketing, ...
    method_name TEXT NOT NULL,                  -- update, write, ...
    task_type TEXT,                             -- forecast.binary, ticketing.tdd
    model_name TEXT NOT NULL,
    quality_score REAL,                         -- per-skill metric (Brier for forecast, etc.)
    cost_usd REAL,
    latency_ms INTEGER,
    program_id TEXT,
    observed_at INTEGER NOT NULL
);

CREATE INDEX idx_skillset_skill_model ON mneme_skillset_observations(skill_namespace, method_name, model_name);
CREATE INDEX idx_skillset_observed_at ON mneme_skillset_observations(observed_at);
```

API:

```rust
impl SkillsetsStore {
    pub async fn record(&self, observation: SkillsetObservation) -> Result<(), StorageError>;
    pub async fn recommend(&self, skill: &str, method: &str, task_type: Option<&str>) -> Option<String>;
    // Returns the model name with the best quality_score / cost_usd ratio for this skill+task combo,
    // or None if not enough observations to recommend confidently.
}
```

Recommendation rules:
- Need ≥5 observations per (skill, method, model) combination before considering it.
- Score = mean(quality_score) / mean(cost_usd). Pick the model with the highest score.
- Cold start: return None; skill uses its documented default.

## Risks

| Risk | Mitigation |
|------|-----------|
| Adding model overrides at every layer means every method signature grows | Keep them all `Option<String>` with sensible defaults; existing callers don't have to change |
| Skillsets registry recommendations early-on are unstable (cold start) | Cold-start threshold of 5 observations per combo; below threshold, recommend None |
| `model_per_trial` invites N-model variance that confounds aggregation | Document; suggest only using for explicit comparison runs (BLFX-13's Sonnet-vs-Opus side-by-side) |
| Per-step model swapping breaks claudecode session continuity (each `claudecode.create` is a new session) | Solved by allowing model override on `chat()` call rather than at session level — needs claudecode change, separate ticket |

## What must NOT change

- Existing `Forecast`, `Ticketing`, `Planning`, etc. constructors (just add new optional params).
- Existing `claudecode.create`'s model parameter behavior.
- Calibration store schema (this is a separate skillsets table).

## Acceptance criteria

1. `forecast.update`, `ticketing.write`, `planning.epic`, `security_review.audit`, `strong_typing.propose` all accept optional `model` parameter.
2. `TrialParams` gets `model` and `model_per_trial` fields; `ClaudeCodeSwarmRuntime::trial` honors them.
3. `SkillsetsStore` implemented in `mneme/skillsets.rs`; SQLite table created at substrate startup.
4. After each `forecast.update` completion, the substrate records a SkillsetObservation (initially with quality_score=NaN until calibrated; cost_usd and latency_ms are immediate).
5. When `forecast.update` is called with `model: None`, it queries `SkillsetsStore.recommend("forecast", "update", task_type)` first, falling back to the default if None.
6. Live test: invoke `forecast.update --model opus` on the running substrate; verify the parent session is created with opus; verify the resulting program manifest records model=opus.
7. Live test: after 5+ forecast.update calls, `programs.list` includes a "recommended_model" derivation OR a separate `skillsets list` method shows current recommendations.
8. `cargo test --lib` passes green.

## Completion

Tests pass + the two live scenarios work; status flips to `Complete` in the same commit as the code.

## Notes on per-step routing (BLFX-4 dependency)

The per-step layer (Layer 3) only matters once the iterative loop is in place. Until then, "model choice at all steps" effectively means "at the trial step." The Layer 3 work can land in BLFX-4's body or as a follow-up; this ticket scopes Layers 1, 2, and 4.
