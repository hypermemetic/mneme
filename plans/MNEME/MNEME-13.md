---
id: MNEME-13
title: "`swarm.sequential` — sequential update with prior-as-context"
status: Pending
type: implementation
blocked_by: [MNEME-8]
unlocks: []
confidence: medium
---

## Problem

`forecast.update` (MNEME-6) implements its own sequential-update path inside the forecast activation. Other skills (e.g., a future "spike sequence" or a multi-step planning revision) need the same shape: `(prior, new_evidence) → new_state`. Promote it to a layer-1 primitive.

## Context

This is a refactor that extracts the sequential-update pattern from MNEME-6 once a second consumer asks for it. Until then, deferring keeps layer-1 minimal.

## Evidence

This ticket is intentionally deferred to phase 3, gated on a real second consumer. Premature abstraction is the failure mode here — extracting the pattern when only `forecast` uses it adds a layer that pays for itself only when a second skill arrives.

Confidence is `medium` because the API is straightforward but the extraction may surface coupling between forecast and the would-be primitive that's hard to anticipate without the second consumer driving the design.

## Required behavior

`swarm.sequential` method:

```
swarm.sequential(
    parent_session: String,
    prior_value: Value,           // arbitrary, opaque to swarm
    prior_summary: String,        // BLF carry-forward statistic
    new_evidence: String,
    response_schema: Value,
    timeout_seconds: u32,
) -> Stream<SequentialEvent>
```

Behavior: forks parent_session, prepends a system message with `prior_value` + `prior_summary`, sends `new_evidence`, awaits typed response via `respond`. Returns `(new_value, new_summary)`.

## Acceptance criteria

1. `swarm.sequential` registered.
2. Refactor `forecast.update` to use it; existing forecast tests still pass.
3. Integration test exercises `swarm.sequential` directly without forecast: prior value `0.3`, prior summary "no evidence yet", new evidence "strong contrary signal", schema `{value: number}`. Asserts response value differs from prior by at least 0.05 (some update happened).
4. `cargo build` and `cargo test` pass green.

## Completion

Test passes; status flips to `Complete` in the same commit.
