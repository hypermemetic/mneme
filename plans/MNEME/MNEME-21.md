---
id: MNEME-21
title: "Bug: ensure_parent_session doesn't recreate when SKILL.md changes"
status: Pending
type: implementation
blocked_by: []
unlocks: []
confidence: high
severity: Medium
---

## Problem

`Forecast.update` calls `ClaudeCodeSwarmRuntime::ensure_parent_session`, which checks if a session with the given name exists. If it does, the function returns Ok(()) without verifying that the existing session's system prompt matches the current `FORECAST_SKILL_MD` baked into the binary.

Discovered live during BLFX-2 verification (see `plans/BLFX/results/bench-001-K5-baseline.md` and the followup K=3 run on 2026-04-26):

1. Substrate restarted with the new BLFX-2 SKILL.md (structured belief state demanded).
2. `forecast.update` called with `--parent-session forecast-fresh-session`.
3. `ensure_parent_session` saw the session existed (created in the prior substrate run with the OLD SKILL.md as system prompt) and returned Ok(()).
4. Trials forked from this session inherited the OLD prompt and produced legacy `{p, summary}` responses, not the new structured shape.
5. Aggregation merged trials successfully (defaults handle missing fields) but `evidence_for / evidence_against / open_questions` were all empty in the artifact.

The artifact at `programs/5740f1fb-8c1b-40ea-b82c-7af3ede70450/artifact.json` shows this clearly: probability and confidence populated; structured fields all empty.

## Context

`ensure_parent_session` is in `mneme-substrate/src/mneme/runtime/claudecode_swarm_runtime.rs::ClaudeCodeSwarmRuntime::ensure_parent_session`. The fast path: `claudecode.get(name)` → if exists, return. The slow path: `claudecode.create(...)` with the system prompt.

The fast path is silently wrong when the system prompt has drifted between runs — common during development when SKILL.md is being edited.

## Required behavior

Pick one of (in order of preference):

**Option A: Hash the system prompt into the session name suffix.**
- Internal session name = `{user_provided_name}-skillv{hash16(SKILL.md)}`.
- Different SKILL.md → different name → automatic recreate.
- User sees old session as orphaned (could be cleaned up later).

**Option B: Compare system prompts at session lookup.**
- `claudecode.get` returns the existing system prompt; `ensure_parent_session` compares; mismatch → log warning, recreate (deleting the old).
- Cleaner from the user's perspective (one session name, always current).
- More state mutation; need careful handling of in-flight forks.

**Option C: Force-recreate flag.**
- `ensure_parent_session` accepts `force_recreate: bool`; caller (Forecast.update) sets it true on first call after substrate boot, false otherwise.
- Punts the policy decision to the caller; doesn't really solve the root issue.

Recommend **A** for simplicity. The implementation is one helper function (`hash_skill_md`) and a name suffix.

## Acceptance criteria

1. `ensure_parent_session` derives a deterministic name suffix from the system prompt content.
2. Live test: launch substrate with SKILL.md A, fire `forecast.update` against `forecast-fresh-session`, verify trials produce structured fields. Then edit SKILL.md (change one word), rebuild substrate, restart, fire `forecast.update` against the same `forecast-fresh-session`, verify a NEW underlying session was created (different suffix) and trials use the new prompt.
2. Old session names with the suffix-pattern are documented as auto-managed; users shouldn't need to know about the hash.
3. `cargo test --lib` passes.

## Completion

Tests pass + live verification per above; status flips to `Complete`.
