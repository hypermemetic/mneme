---
id: MNEME-14
title: "`swarm.race` — first-success-wins for spike chains"
status: Pending
type: implementation
blocked_by: [MNEME-8]
unlocks: []
confidence: medium
---

## Problem

The planning skill's spike chain pattern (S-01 → S-02 → S-03, stop at first success) needs a primitive that runs alternatives in order, returning as soon as one passes. Without it, callers wire this pattern by hand and the recording loses the "stopped at attempt N" structure.

## Context

Like MNEME-13, this is deferred to phase 3 until a real consumer (planning's spike orchestration, MNEME-10's deeper integration) asks for it.

## Evidence

Spike chains as a primitive are valuable when planning evolves to actually execute spikes (rather than just write spike tickets). Phase 1 + 2 ship without this; planning produces spike *tickets* and humans run them. Phase 3 closes the loop: planning produces spike tickets AND can execute them via `swarm.race`.

## Required behavior

`swarm.race` method:

```
swarm.race(
    parent_session: String,
    alternatives: Vec<Alternative>,    // ordered list of (prompt, predicate)
    response_schema: Value,
    per_alt_timeout_seconds: u32,
) -> Stream<RaceEvent>
```

Each `Alternative`: `{ prompt: String, pass_predicate: String }`. The pass_predicate is a simple expression evaluated against the typed response (e.g., `"data.found_in_pdf == true"`).

Behavior: try alternatives in order; for each, fork parent_session, send prompt, await typed response, evaluate predicate. Return first that passes. If all fail, return all attempts so the caller can replan.

## Acceptance criteria

1. `swarm.race` registered.
2. Integration test: 3 alternatives, second one designed to pass. Asserts the second passes, the third is never attempted.
3. Integration test: all alternatives fail. Asserts all 3 attempts are returned with their failures.
4. `cargo build` and `cargo test` pass green.

## Completion

Test passes; status flips to `Complete` in the same commit.
