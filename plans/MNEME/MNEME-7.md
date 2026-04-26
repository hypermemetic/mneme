---
id: MNEME-7
title: "Program reification — recording & manifest writer"
status: Pending
type: implementation
blocked_by: [MNEME-2, MNEME-6]
unlocks: [MNEME-8]
confidence: medium
---

## Problem

The MVP works only if every skill invocation produces the program directory contracted in MNEME-2. Without recording, skills run but leave no audit trail; replay, calibration, and child-program linking become impossible. This ticket integrates the writer into the orchestration path so recording is implicit, not opt-in.

## Context

- MNEME-2 pins the directory layout and manifest format.
- MNEME-6 (forecast) writes its own `belief_state.md` and `updates/`, but doesn't write the per-program `manifest.json`, `trace.jsonl`, or `sessions/` exports.
- `claudecode.sessions_export(session_id)` produces the JSON file the writer drops into `sessions/`.
- Every `swarm.*` invocation should produce one `trace.jsonl` line; the writer is invoked from inside the swarm activation.

## Evidence

Recording lives at the orchestration layer (inside `swarm`) rather than per-skill because:

1. Every skill uses `swarm.*` — instrumenting once covers all skills.
2. Skills shouldn't have to know about the program directory format; they get a `program_id` they pass to swarm calls and the recording happens transparently.
3. Trace coherence requires monotonic seq numbers per program; centralizing the writer prevents skills from clobbering each other.

The implementation choice is filesystem-direct (no broker, no queue) because programs are append-only and a single program is single-threaded — there's no contention to mediate.

Confidence is `medium`: the writer is straightforward, but integrating it cleanly across the swarm methods without coupling them to the writer's lifetime is the part that may need a refactor pass.

## Required behavior

A `recorder` module in mneme:

```
struct ProgramRecorder { /* program_id, root_path, seq_counter, ... */ }

impl ProgramRecorder {
    fn open(program_id: ProgramId, entry_skill: &str, inputs: &Value) -> Result<Self>;
    fn next_seq(&mut self) -> u32;
    fn write_trace(&mut self, op: &str, args: &Value, child_sessions: &[String], outcome: TraceOutcome, duration: Duration) -> Result<()>;
    fn export_session(&mut self, session_id: &str, claudecode_handle: &Hub) -> Result<()>;
    fn write_artifact<T: Serialize>(&mut self, value: &T, schema_version: &str) -> Result<()>;
    fn write_error(&mut self, kind: &str, message: &str, stage: &str) -> Result<()>;
    fn close(self, status: ProgramStatus) -> Result<()>;   // updates manifest.finished_at + status
}
```

Behavior:

| Scenario | Expected outcome |
|----------|-----------------|
| `swarm.trial` invoked with a `program_id` | After all trials terminate, the recorder writes one `trace.jsonl` line and exports each trial's session into `sessions/` |
| Skill invoked without a program_id (e.g., direct test call) | Writer is a no-op; skill returns its result without filesystem side effects |
| Two concurrent swarm calls within the same program | Trace lines have monotonically increasing seq, no missing values, no duplicates |
| Program errors mid-flight | `close(Failed)` writes `error.json` and updates manifest.status; partial sessions/ stays as-is for forensics |
| Loopback child program | Parent's recorder gets called with the child's `program_id`; child's directory is created at `programs/<parent>/skills/<child>/`. Parent's trace records the child program id. |

## Risks

| Risk | Mitigation |
|------|-----------|
| File I/O during a long trial blocks the trial path | Recorder calls are async, writes are batched per trial completion (not per ChatEvent) |
| `sessions_export` may fail if the session was deleted (e.g., timeout cleanup) | Recorder logs warning + writes a placeholder `sessions/<id>.json` with `{"_warning": "export_failed"}` |
| Race between two skills sharing a swarm activation instance | Each program's recorder is a dedicated instance; the swarm activation looks up the recorder by program_id from a thread-safe map |

## What must NOT change

- `swarm.trial` / `swarm.aggregate` external behavior. Recording is added internally; events emitted to the caller are unchanged.
- The directory layout pinned in MNEME-2.

## Acceptance criteria

1. `mneme/src/recorder.rs` implements `ProgramRecorder` per the API above.
2. `swarm.trial` and `swarm.aggregate` accept an optional `program_id` parameter; when provided, recording happens.
3. Integration test `tests/recorder_layout.rs`: invokes `swarm.trial` with `program_id = "test-001"`, n=2, asserts:
   - `programs/test-001/manifest.json` exists with the right `entry_skill` and `started_at`
   - `programs/test-001/trace.jsonl` has exactly one line with `op: "swarm.trial"` and 2 child session ids
   - `programs/test-001/sessions/` contains 2 `<session_id>.json` files
   - `programs/test-001/manifest.json.status` is `completed`
4. Integration test `tests/recorder_failure.rs`: triggers a swarm.trial that fails (e.g., bad schema); asserts `error.json` is written and `manifest.status == "failed"`.
5. Integration test `tests/recorder_concurrent_seq.rs`: 3 swarm calls in flight against the same program_id; asserts trace seq is `1, 2, 3` with no duplicates or gaps.
6. `cargo build` and `cargo test` pass green for the `mneme` crate.

## Completion

Implementor runs the three recorder tests, all pass; status flips to `Complete` in the same commit as the code.
