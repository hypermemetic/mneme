---
id: MNEME-2
title: "Define program directory layout & manifest format"
status: Pending
type: analysis
blocked_by: []
unlocks: [MNEME-7]
confidence: high
---

## Problem

The harness records every skill invocation as a directory artifact ("a program"). Without a fixed layout and manifest format, downstream tools (replay, audit, child-program linking, calibration loop) can't read program artifacts reliably. This ticket pins the contract.

## Context

- `claudecode.sessions_export` produces a JSON file per session — those drop into `programs/<id>/sessions/`.
- A skill invocation may spawn child skill invocations (via loopback). Each child is its own program; the parent program references it.
- The artifact (the typed result of the entry skill) needs to be readable without parsing the trace.

## Evidence

The four-layer architecture (epic MNEME-1) treats programs as the residue of execution. Three downstream consumers depend on this contract being stable:

1. **MNEME-7** (recording integration) writes the manifest and trace; the writer must know the format.
2. **MNEME-8** (harness binary) reads the artifact to print results to stdout; needs to know where the artifact lives in the directory.
3. **MNEME-15** (calibration loop) walks resolved programs to update Platt parameters; needs `manifest.json` to identify forecast programs and resolution dates.

Splitting layout from implementation lets MNEME-3..6 proceed in parallel — they only need to know the *shape* of what gets recorded, not the writer.

The contract is intentionally filesystem-native (not a sqlite db) because (a) programs are append-once-write-never-modify, (b) `git log` over `programs/` becomes free audit history, (c) human inspection with `tree` and `cat` is the lowest-friction debugging tool.

## Required behavior

The `programs/<program_id>/` directory MUST contain:

| Path | Format | Required | Description |
|------|--------|----------|-------------|
| `manifest.json` | JSON | yes | Program metadata (see below) |
| `artifact.json` | JSON | yes if completed | The skill's typed return value, schema-validated |
| `trace.jsonl` | JSON Lines | yes | One line per Layer-1 call (`swarm.*`) |
| `sessions/<session_id>.json` | JSON | yes for completed | One file per `claudecode` session created during the program |
| `skills/<child_program_id>/` | directory | optional | One subdirectory per child program (recursive) |
| `error.json` | JSON | yes if failed | `{ "kind": "...", "message": "...", "stage": "..." }` |

`manifest.json` shape:

| Field | Type | Description |
|-------|------|-------------|
| `program_id` | string (UUID) | Globally unique |
| `parent_program_id` | string \| null | Set if this program was spawned by another via loopback |
| `entry_skill` | string | e.g., `forecast.update` |
| `inputs` | object | The arguments passed to the skill method (JSON) |
| `inputs_schema_version` | string | Schema version of the inputs (semver) |
| `started_at` | ISO 8601 string | UTC |
| `finished_at` | ISO 8601 string \| null | UTC; null if still running |
| `status` | enum | `running` \| `completed` \| `failed` |
| `artifact_schema_version` | string \| null | Set when artifact is written |
| `substrate_version` | string | The mneme-substrate semver that ran the program |
| `mneme_version` | string | The harness binary semver |

`trace.jsonl` line shape (one per `swarm.*` call):

| Field | Type | Description |
|-------|------|-------------|
| `seq` | integer | Monotonic from 1 |
| `at` | ISO 8601 | UTC |
| `op` | enum | `swarm.trial` \| `swarm.aggregate` \| `swarm.sequential` \| `swarm.race` |
| `args_summary` | object | Skill-defined; small enough to read inline |
| `child_session_ids` | string[] | claudecode session ids spawned by this op |
| `child_program_ids` | string[] | child programs spawned (loopback) |
| `outcome` | enum | `ok` \| `err` |
| `duration_ms` | integer | Wall-clock |

## Risks

| Risk | Mitigation |
|------|-----------|
| Directory layout changes after phase 1, breaking MNEME-15 reads | Pin schema versions; readers must check version + handle older formats |
| Trace becomes too large (a swarm.trial with N=8 trials × 50 events each = 400 entries) | trace.jsonl entries summarize only the swarm op; per-trial events live in sessions/ — kept separate |
| Loopback child programs create unbounded recursion if a skill calls itself | Manifest carries depth field; harness refuses depth > 8 |

## What must NOT change

- The four-layer architecture in MNEME-1's diagram. This ticket is contract-pinning, not architecture revision.
- The pre-existing `claudecode.sessions_export` output format. The harness writes those files into `sessions/` unchanged.

## Acceptance criteria

1. `docs/program-format.md` exists in this repo and describes every field in the tables above with examples.
2. `docs/program-format.md` includes one fully-worked example: a `forecast.update` program with N=3 trials, showing every file's contents.
3. The format is reviewed against `claudecode.sessions_export`'s actual output (sample one, paste a few lines into the doc) so the `sessions/<id>.json` shape is real, not invented.
4. All schema-version fields default to `"0.1.0"` for the MVP; the doc states the bump policy.
5. No code is written by this ticket — it is an analysis/contract ticket only.

## Completion

Implementor delivers `docs/program-format.md`, flips status to `Complete` in this ticket, and commits both in one change.

## Implementation status (2026-04-26 autonomous session)

The data model is implemented in `mneme-substrate/src/mneme/program/` (manifest.rs, trace.rs, directory.rs, id.rs, mod.rs). The schema-version field defaults are wired (`MANIFEST_SCHEMA_VERSION = "0.1.0"`). The `docs/program-format.md` companion doc is NOT yet written — that's the remaining work for this ticket. The Rust code is the source of truth for the format until the doc catches up; readers can `cargo doc --open` the manifest module for now. Architecture overview at `mneme-substrate/docs/architecture/16669551678743899647_mneme-architecture.md`.
