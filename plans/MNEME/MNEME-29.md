---
id: MNEME-29
title: "LLM capability library + JSON-cleanup recovery on parse failures"
status: Ready
type: implementation
blocked_by: []
unlocks: [BLFX-14]
confidence: high
severity: Medium
---

## Problem

Two related gaps surfaced in bench-005 + bench-006:

1. **Schema-violation failures abort entire trials.** 5/6 of bench-005's
   failures (and several in bench-006 in flight) are the model emitting
   `evidence_for: ["bare string"]` instead of
   `evidence_for: [{"claim": "...", "weight": 0.5}]`. A lenient
   `EvidenceItem` deserializer (committed) handles this specific case,
   but the *general* pattern — the agent emits malformed JSON,
   `parse_step` returns Err, the trial dies — recurs on every new
   schema field we add. This isn't an EvidenceItem problem; it's a
   systemic recovery problem.

2. **Every model call uses Sonnet** even when a cheaper model would do.
   Search-worker subsessions, JSON cleanup, summarization — these are
   structural tasks where Haiku is plenty. We're spending ~10× the
   token cost on tasks that don't need it. And we have no abstraction
   that lets the iterative loop ask for "a model good at JSON cleanup"
   without hardcoding which model.

The fix is one substrate-side abstraction (capability registry) plus
one wire-up (parse-failure recovery via cleanup capability).

## Context

- Existing models live in `mneme-substrate/src/activations/claudecode/`
  as the `Model` enum (`Opus | Sonnet | Haiku`). Today the swarm runtime
  hardcodes Sonnet for all trials and search-workers.
- The `parse_step` function in
  `mneme-substrate/src/activations/forecast/agent_loop.rs` returns
  `Result<ParsedStep, ParseError>`. The iterative loop in
  `mneme-substrate/src/activations/forecast/iterative_loop.rs::iterative_trial`
  propagates `ParseError` as `LoopError::Parse`, which aborts the trial.
- The action+belief schema for the iterative loop is documented in
  `mneme-substrate/src/mneme/runtime/claudecode_step_driver.rs::ITERATIVE_FORMAT_CONTRACT`
  (the per-step prompt's JSON shape spec).
- BLFX-14 ("model-as-first-class-choice at every step") is the
  algorithmic-side ticket that consumes what this builds. This ticket
  lays the substrate; BLFX-14 plumbs it through the iterative loop's
  per-step decision logic.

## Required behavior

Build `mneme-substrate/src/mneme/capabilities/` with three pieces:

**1. Capability + Registry types.**

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum CostTier { Cheap, Standard, Premium }

#[derive(Debug, Clone)]
pub struct Capability {
    pub name: String,           // e.g. "json_cleanup"
    pub model: Model,           // Haiku | Sonnet | Opus
    pub cost_tier: CostTier,
    pub max_tokens: u32,
}

pub struct CapabilityRegistry { /* private map */ }

impl CapabilityRegistry {
    pub fn default_substrate() -> Self;  // ships with the standard set
    pub fn get(&self, name: &str) -> Option<&Capability>;
    pub fn register(&mut self, cap: Capability);
}
```

The default registry ships with at minimum:

| name | model | cost | use |
|---|---|---|---|
| `reasoning_default` | Sonnet | Standard | the iterative loop's main reasoner |
| `reasoning_strong` | Opus | Premium | future BLFX-14 final-step commitment |
| `json_cleanup` | Haiku | Cheap | recover malformed JSON output |
| `summarize_short` | Haiku | Cheap | future search-worker summarization |

**2. Invocation API that wraps `claudecode.chat`.** Given a capability
name + a prompt, create an ephemeral session, fire one chat, return
the response text. The capability decides the model; the substrate
takes care of session lifecycle.

```rust
impl CapabilityRegistry {
    pub async fn invoke<P: HubContext + 'static>(
        &self,
        claudecode: Arc<ClaudeCode<P>>,
        capability_name: &str,
        prompt: String,
        working_dir: String,
    ) -> Result<String, CapabilityError>;
}
```

Errors are explicit (unknown capability name, chat error, timeout).

**3. Wire JSON-cleanup recovery into `parse_step` failures.** In
`iterative_trial`, when `parse_step` returns Err on a step's raw
output:

a. Construct a cleanup prompt:
```
The following text was supposed to be JSON matching this schema:

<schema_json>

The parser returned this error: <error_msg>

Output ONLY the corrected JSON object — no prose, no markdown fence,
no apology. Single line is fine. If the structure is unrecoverable,
output {}.

---

<raw text from the agent>
```

b. Invoke the `json_cleanup` capability with the prompt.

c. Re-run `parse_step` on the cleaned output.

d. If `parse_step` succeeds, log
   `tracing::info!("parse_step recovered via json_cleanup")` and continue
   with the recovered ParsedStep.

e. If it fails again, propagate the original error. **One retry max.**
   Don't loop.

The recovery only fires on `ParseError::BadJson`,
`ParseError::MissingField`, and `ParseError::InvalidAction` /
`InvalidBelief`. Don't fire on `NoStepBlock` (the model didn't even
emit a json fence — cleanup can't fix that).

## Acceptance criteria

1. `mneme/capabilities/` module exists with `Capability`, `CostTier`,
   `CapabilityRegistry`, `CapabilityError`.
2. `CapabilityRegistry::default_substrate()` ships at least the four
   capabilities above.
3. Unit tests: `default_substrate has json_cleanup`, `invoke unknown
   capability errors`, `invoke routes to the configured model`
   (mockable).
4. `iterative_trial` consults the registry on `parse_step` failure;
   on success, logs the recovery and continues.
5. New integration test in `iterative_loop`: a `QueueStepDriver` that
   first emits malformed JSON, then a recovery-capability mock that
   returns clean JSON; assert the loop succeeds and history records
   the step.
6. No changes to `forecast.update`'s public method signature. The
   recovery is transparent.
7. `cargo test --lib` passes.
8. Live verification: re-run bench-006's failed-question subset with
   the recovery path active. ≥80% of the failures should now succeed
   (we expect the bare-string-evidence pattern to be fixed already by
   the lenient EvidenceItem deserializer; recovery covers the next
   tier of failures — wrong-shaped action types, missing fields, etc).

## Out of scope

- Per-step / per-trial model selection in the iterative loop (BLFX-14).
- Cost tracking + per-capability budgets (later).
- Async streaming (cleanup is one-shot — single chat call, single
  response).
- Multi-tier recovery (cleanup → re-cleanup → … forever). One retry
  is the rule.
- Replacing the search-worker session's hardcoded Sonnet (BLFX-14 territory).

## Completion

Tests pass + bench-006-style failure subset succeeds via recovery
path. Status → `Complete`. BLFX-14 gets a one-line update noting the
substrate is ready for per-step model selection.
