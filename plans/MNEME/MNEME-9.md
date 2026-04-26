---
id: MNEME-9
title: "Port `ticketing` as activation"
status: Pending
type: implementation
blocked_by: [MNEME-8]
unlocks: []
confidence: medium
---

## Problem

The `ticketing` skill exists as documentation (`hypermemetic/skills/skills/ticketing/SKILL.md`). Port it to a Plexus activation so callers can invoke `ticketing.write` from mneme or via loopback from another skill (e.g., `security_review` filing tickets for findings).

## Context

The ticketing SKILL.md is already calibrated to the BLF carry-forward principle (it has the `## Evidence` section + `confidence:` frontmatter field). The port wraps the prompt; the SKILL.md content is the system prompt baked via `include_str!`.

## Evidence

Each skill port is the same shape: load SKILL.md → register activation with one or two methods → use `respond` for typed output → use `swarm.trial` if multi-trial is desired. Doing them as separate tickets (rather than one big "port all skills" ticket) lets implementors take them in parallel and lets phase-3 calibration data identify which skills produce reliable typed outputs and which don't.

`ticketing.write` benefits from `swarm.trial` with N=2 and `MajorityEnum` aggregation on whether the contract is single-decision (passing the meta-acceptance criteria) — but MVP can ship single-trial and add multi-trial after observing failure modes.

## Required behavior

`ticketing` activation, namespace `ticketing`, version `0.1.0`. Methods:

```
ticketing.write(
    epic: String,                 // EPIC prefix, e.g., "AUTH"
    problem: String,              // one-paragraph description
    upstream_tickets: Vec<String>,
    downstream_consumers: Vec<String>,
    domain_context: Option<String>,
    confidence: Option<String>,   // high | medium | low; default medium
) -> Stream<WriteEvent>
```

Output (`WriteEvent::Completed`) includes the typed `Ticket`:

| Field | Type |
|-------|------|
| `id` | string (`<EPIC>-<N>` allocated by reading existing files) |
| `frontmatter` | object (matches the ticketing skill's frontmatter spec) |
| `body` | string (markdown body conforming to the skill's section order) |
| `meta_checks` | object (each meta-acceptance-criterion item: passed/failed) |
| `file_path` | string (where the ticket would be written) |

The activation does NOT write the file to disk on its own — mneme or the caller decides. This keeps the activation pure (no FS side effects beyond program recording).

## Acceptance criteria

1. `ticketing` activation registered.
2. `mneme run ticketing.write --epic AUTH --problem "..." --upstream-tickets AUTH-2 --downstream-consumers AUTH-7` returns a typed Ticket with all meta-checks passing.
3. The returned `body` includes `## Evidence` section, observable acceptance criteria, and no implementation prescriptions.
4. Integration test asserts the meta-checks accurately reflect the body (e.g., remove the `## Evidence` section from the body and the `evidence_present` check fails).
5. `cargo build` and `cargo test` pass green.

## Completion

Test passes; status flips to `Complete` in the same commit.
