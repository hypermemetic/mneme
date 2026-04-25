---
id: MNEME-10
title: "Port `planning` as activation"
status: Pending
type: implementation
blocked_by: [MNEME-8]
unlocks: []
confidence: medium
---

## Problem

The `planning` skill exists as documentation. Port it to a Plexus activation so callers can invoke `planning.epic` to break a goal into a DAG of tickets, with spikes for unknowns.

## Context

The planning SKILL.md already specifies output shape (epic overview + per-ticket frontmatter + risks→spikes mapping). The port wraps the prompt and uses `ticketing.write` (via loopback) to materialize each ticket.

## Evidence

`planning` is the first skill that genuinely composes — it must call `ticketing.write` once per ticket in the resulting DAG. This validates the loopback-based skill composition pattern from the architecture epic. If MNEME-3's `respond` tool registration uses program-id-suffixed names, concurrent loopback calls from one planning invocation must not collide; this ticket exercises that.

Confidence is `medium` because the composition path is exercised here for the first time, even though planning's own logic is straightforward.

## Required behavior

`planning` activation, namespace `planning`, version `0.1.0`. Methods:

```
planning.epic(
    goal: String,
    constraints: Vec<String>,
    epic_prefix: String,           // e.g., "AUTH"
    domain_context: Option<String>,
    existing_work: Option<String>,
) -> Stream<EpicEvent>
```

Output (`EpicEvent::Completed`):

| Field | Type |
|-------|------|
| `epic_id` | string (e.g., `AUTH-1`) |
| `overview` | object (matches the planning skill's epic overview format) |
| `tickets` | Vec<Ticket> (each from `ticketing.write` via loopback) |
| `dag_check` | object (parallelism opportunities, file-boundary collisions, etc.) |
| `child_program_ids` | Vec<String> (one per ticket.write loopback call) |

## Acceptance criteria

1. `planning` activation registered.
2. `mneme run planning.epic --goal "Thread auth context through Plexus RPC" --epic-prefix AUTH` returns a typed result with at least 3 tickets.
3. Each ticket in the result has a corresponding child program directory under the epic's program.
4. The `dag_check` reports no concurrent siblings writing the same file.
5. Integration test verifies that loopback child programs link to the parent in `manifest.parent_program_id`.
6. `cargo build` and `cargo test` pass green.

## Completion

Test passes; status flips to `Complete` in the same commit.
