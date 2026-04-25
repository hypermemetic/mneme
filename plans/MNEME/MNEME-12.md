---
id: MNEME-12
title: "Port `strong_typing` as activation"
status: Pending
type: implementation
blocked_by: [MNEME-8]
unlocks: []
confidence: high
---

## Problem

The `strong-typing` skill exists as documentation. Port it to a Plexus activation so callers can invoke `strong_typing.propose` to produce a newtype refactor plan for a codebase.

## Context

The strong-typing SKILL.md is the smallest and most mechanical of the skills — its inputs and outputs are well-defined (inventory of bare-typed domain values → list of newtype proposals). Single-trial is sufficient.

## Evidence

This is the simplest port. Confidence is `high` because (a) the skill output is mostly tabular, (b) no multi-trial aggregation is needed (one good pass suffices), (c) no file writes — the activation produces a proposal, the human applies it. It's a useful sanity check that simple skills don't get lost in the harness machinery.

## Required behavior

`strong_typing` activation, namespace `strong_typing`, version `0.1.0`. Methods:

```
strong_typing.propose(
    codebase_root: String,
    focus_modules: Option<Vec<String>>,    // narrow the inventory; default = all
) -> Stream<ProposeEvent>
```

Output (`ProposeEvent::Completed`):

| Field | Type |
|-------|------|
| `proposals` | Vec<NewtypeProposal> |
| `boundary_check` | object (which proposed types validate at the wire boundary; which need re-validation deeper) |

`NewtypeProposal`: `{ name, current_type, locations: [file:line], rationale, estimated_call_sites }`.

## Acceptance criteria

1. `strong_typing` activation registered.
2. `mneme run strong_typing.propose --codebase-root /path` returns a proposals list with at minimum one proposal per distinct domain identifier currently typed as `String`.
3. Each proposal cites file:line locations that exist and grep-match the bare type.
4. Integration test against a small fixture with 3 known-bare-typed identifiers: asserts all 3 are proposed with correct names.
5. `cargo build` and `cargo test` pass green.

## Completion

Test passes; status flips to `Complete` in the same commit.
