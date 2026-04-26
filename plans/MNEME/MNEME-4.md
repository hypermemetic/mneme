---
id: MNEME-4
title: "`swarm.trial` — fan-out via fork"
status: Pending
type: implementation
blocked_by: [MNEME-3, MNEME-S02, MNEME-S03]
unlocks: [MNEME-5, MNEME-6]
confidence: medium
---

## Problem

BLF multi-trial — and any other "ask N independent Claudes the same question" pattern — needs a primitive that takes a parent session, a prompt, a response schema, and `N`, and returns N structured responses with their per-trial session ids. No such primitive exists; callers would have to manage N forks, N async chats, and aggregation by hand.

## Context

- `claudecode.fork(name, new_name)` branches a session at its current head. A fresh fork inherits the parent's working dir, model, system prompt, and tree position.
- `claudecode.chat_async(name, prompt, ephemeral?)` spawns a non-blocking chat that returns a stream id; results polled via `claudecode.poll(stream_id)`.
- MNEME-3 provides per-skill `respond` tool registration that yields typed responses.
- MNEME-S02 establishes whether forks under a shared system prompt produce independent reasoning paths (the BLF independence assumption).

## Evidence

The fork-based design exploits an existing primitive (`claudecode.fork`) rather than spinning up N independent sessions, because:

1. Fork preserves the parent's prepared context (system prompt, prior turns) — so trials start from the same belief state, which is the BLF "shared prior" assumption.
2. The parent's tree position is the explicit divergence point in the arbor; replay and audit are natural.
3. Per-trial sessions can be exported individually into `programs/<id>/sessions/`, matching the recording layout in MNEME-2.

If S02 reports that fork-based trials are too correlated, this ticket adopts S02's fallback: per-trial system-prompt diversification (e.g., "you are reviewer #i of N"). Confidence is `medium` because the design is cheap to implement but its *value* depends on S02 confirming independence.

`swarm.trial` is the *only* layer-1 primitive callers should use to create per-trial sessions; this concentrates fork orchestration in one place so MNEME-7's recording and MNEME-S03's parallel-fork bound have one consumer to track.

## Required behavior

`swarm` activation, namespace `swarm`, version `0.1.0`. New method:

```
swarm.trial(
    parent_session: String,        // existing claudecode session
    prompt: String,                // sent to each trial
    response_schema: Value,        // JSON Schema for typed response (per MNEME-3)
    n: u8,                         // 1..=16
    diversify: Option<String>,     // optional template, %i replaced with trial index
    timeout_seconds: u32,          // per-trial; default 120
) -> Stream<TrialEvent>
```

Events:

| Variant | Fields | When |
|---------|--------|------|
| `Started` | `trial_count: u8`, `program_id: String` | Immediately after validation |
| `TrialStarted` | `trial_index: u8`, `session_id: String` | Each fork created |
| `TrialCompleted` | `trial_index: u8`, `response: Value`, `duration_ms: u64` | Each trial's `respond` call returns a valid payload |
| `TrialFailed` | `trial_index: u8`, `error: String`, `last_payload: Option<Value>` | Trial timed out or exhausted respond retries |
| `Completed` | `successes: u8`, `failures: u8` | All trials reached terminal state |

Behavior:

| Scenario | Expected outcome |
|----------|-----------------|
| `n = 1` | Single fork; semantically identical to a one-call wrapper. Recording still happens. |
| `n > 16` | Validation error. Cap is from MNEME-S03's findings; if S03 reports a lower bound, this constant is bumped down. |
| One trial fails, others succeed | `Completed` event fires after all reach terminal; consumer aggregates over `TrialCompleted` events. |
| All trials fail | `Completed` fires with `successes: 0`; consumer decides whether that's an error. `swarm.trial` itself does not error in this case. |
| `diversify` is `Some("Reviewer %i of {N}")` | Each fork's first turn includes the diversification clause as a prepended user message. |

## Risks

| Risk | Mitigation |
|------|-----------|
| Forks correlate too strongly (S02 fail) | Use the `diversify` parameter; if still correlated, document and ship N=1 default for forecast |
| Substrate degrades above N=8 (S03 fail) | Cap N from S03's findings; expose the cap via `swarm.limits` method for callers to discover |
| Per-trial respond tool registration leaks if a trial panics | Tool deregistration on `Drop`; integration test exercises panic-in-trial path |
| Async polling adds latency vs. fully concurrent | Use `chat_async` + `poll` with 100ms intervals; alternative is concurrent `chat` calls (blocking), which may be simpler — pick based on S03 latency data |

## What must NOT change

- The `claudecode` activation. This ticket consumes `fork`, `chat_async`, `poll` as-is.
- The recording contract from MNEME-2. Trials are recorded under the existing `sessions/` layout.

## Acceptance criteria

1. `swarm` activation registered in `mneme/src/swarm/mod.rs` and added to a builder following the standard plexus pattern.
2. Integration test `tests/swarm_trial_basic.rs`: parent session with system prompt "you are a forecaster", prompt "P that the sun rises tomorrow?", n=3, schema for `{probability: number}`. Asserts 3 `TrialCompleted` events, all with `probability >= 0.99`.
3. Integration test `tests/swarm_trial_diversify.rs`: same as above with `diversify: Some("Take perspective #%i: bull, bear, base")`, n=3. Asserts the trial responses' reasoning text contains the diversification clauses.
4. Integration test `tests/swarm_trial_validation.rs`: n=20 returns `Err(InvalidParam)`.
5. `cargo build` and `cargo test` pass green for the `mneme` crate.

## Completion

Implementor runs the three swarm.trial tests, all pass; status flips to `Complete` in the same commit as the code.
