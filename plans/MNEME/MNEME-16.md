---
id: MNEME-16
title: "`programs.subscribe` — arbor-backed live event stream for a running program"
status: Pending
type: implementation
blocked_by: []
unlocks: []
confidence: medium
---

## Problem

When a `forecast.update` (or any other fire-and-return skill invocation) is in flight, callers have no way to watch what's happening in real time. They can poll `programs.status` for lifecycle transitions, but they can't see the actual reasoning Claude is producing inside each trial until the program closes and the artifact is written.

The user's framing: *we should be able to tune into the stream at any point; we fully save the outputs as they happen. tune into the arbor — that way we just get the arbor changes for free. that's the real history of what's happening.*

## Context

- Each `claudecode` session writes its conversation events (UserMessage, AssistantStart, ContentText, ContentToolUse, ContentThinking, AssistantComplete, etc.) to an Arbor tree via `claudecode/storage.rs`.
- A `forecast.update` invocation forks the parent session N ways; all N trials share the same arbor tree (forks are branches in the tree, not separate trees).
- The substrate's `arbor` activation already exposes tree state; what's missing is a per-program subscription that filters arbor writes to those belonging to the program's child sessions.
- `mneme-substrate/src/mneme/runtime/session_attribution.rs` already maps session_id → program_id (Phase 2 stub; not yet populated by `claudecode.create` / `fork` calls).

## Evidence

The user flagged this as the right primitive because:

1. Arbor IS the stream. Every Claude reasoning step writes a node. The data already exists; we just need to expose it.
2. Building a custom event channel (broadcast, etc.) would duplicate what arbor already does and would lose persistence on substrate restart.
3. Subscribers reading arbor get for free: persistence (the events are durable), retroactive reads (subscribe after the program started, replay everything), and the same view across restarts.

The implementation has two ordered prerequisites:
- **Session attribution must work.** When `claudecode.create` or `fork` is called from inside a `swarm.trial` invocation, the resulting `session_id` must be attributed to the calling program. Without this, the subscription has no way to know which arbor tree positions to watch.
- **Arbor needs a watch / subscribe primitive.** Today arbor exposes read methods (get_tree, render_context). It needs either a tail-on-tree primitive or a tokio broadcast channel that emits NodeEvent on every write.

## Required behavior

`programs` activation gains one method:

```
programs.subscribe(
    program_id: String,
    from_position: Option<String>,   // resume from a known arbor position (for catch-up)
) -> Stream<SubscribeEvent>
```

`SubscribeEvent` variants:

| Variant | Fields | When |
|---|---|---|
| `Started` | `program_id, sessions_attributed: u8, current_position: String` | First event, before any backfill |
| `Node` | `session_id, position, role, content_kind, content_excerpt` | One per arbor node (backfill + live) |
| `ProgramClosed` | `status: completed \| failed` | When the program's manifest flips out of `running` |
| `Error` | `message: String` | Subscription error (program missing, etc.) |

Behavior:

| Scenario | Expected outcome |
|----------|-----------------|
| Subscribe to a running program | Backfill all arbor nodes for sessions attributed to the program; then continue streaming as new nodes are written; close stream when program manifest flips out of running |
| Subscribe to an already-completed program | Backfill all nodes; emit ProgramClosed; close stream |
| Subscribe to an unknown program_id | Emit Error and close |
| Subscribe twice to the same program | Both subscriptions get full backfill independently; both stay live until close |
| Program spawns child program via loopback | Child program's nodes are NOT in the parent's subscription (use `programs.subscribe(child_id)` separately); the trace.jsonl line for the loopback IS in the parent's program directory |
| `from_position` provided | Backfill resumes from that arbor position (for clients reconnecting after dropped connection) |

## Risks

| Risk | Mitigation |
|------|-----------|
| Arbor doesn't currently expose a tail-on-write primitive | First-class arbor change OR poll arbor on a 200ms interval per subscription (cheap; arbor reads are sqlite SELECTs) |
| Session attribution is unwired | Phase-1 of this ticket: wire SessionAttribution into ClaudeCodeSwarmRuntime (every fork attributes the resulting session_id) |
| Subscriber count grows unbounded | Subscriptions are per-call; they end when the program closes; substrate doesn't hold them across restarts |
| Backfill of a long-running program (many nodes) overwhelms the client | Default page size 100 nodes; client can pass `from_position` to control resumption |

## What must NOT change

- The arbor activation's existing read methods.
- The `claudecode` activation's session/fork/chat semantics.
- The fire-and-return shape of `forecast.update`.

## Acceptance criteria

1. `programs.subscribe` registered on the existing `programs` activation.
2. `ClaudeCodeSwarmRuntime` populates `SessionAttribution` whenever it forks a session for a trial.
3. Integration test: spawn a `forecast.update` against `DeterministicMockSwarmRuntime` (modified to record a fake arbor write per trial); subscribe to the program; assert receive Started, then N Node events, then ProgramClosed.
4. Integration test against the real ClaudeCode runtime: forecast.update with 2 trials; subscribe; assert at least one ContentText Node event arrives before ProgramClosed (proves backfill+live works end-to-end).
5. `cargo build --bin mneme-substrate` and `cargo test --lib` pass green.

## Completion

Implementor runs the integration tests, all pass; verifies live by `synapse -P 4456 substrate programs subscribe --program-id <id>` while a `forecast.update` is running and observes streaming events; status flips to `Complete` in the same commit as the code.
