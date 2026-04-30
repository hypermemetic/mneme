---
id: MNEME-32
title: "programs.wait — substrate-level block-and-stream for fire-and-return programs"
status: Complete
type: implementation
blocked_by: []
unlocks: []
confidence: high
severity: Medium
forecast:
  hypothesis: "Will programs.wait ship within 24 hours, work end-to-end against the running container, and let us delete scripts/mneme by replacing all its load-bearing commands with synapse natural-CLI invocations?"
  resolution_method: "After implementation, verify: (1) `synapse -P 4456 substrate forecast update --program-id X --new-evidence ... | synapse -P 4456 substrate programs wait` blocks and prints the artifact; (2) `cargo test --lib` green; (3) the `mneme` CLI script can be deleted and the README's quickstart uses synapse only."
  deadline: "2026-05-01T00:00:00Z"
---

## Problem

`forecast.update` is fire-and-return — it spawns the trial+aggregate
work in the background and returns immediately with `Started { program_id }`.
This is the right semantics for the protocol (long-running work shouldn't
hold an RPC call open), but it leaves the obvious "give me the result"
UX as a polling pattern.

The previous fix (committed as `scripts/mneme`) wrapped synapse in a
Python CLI that polls `programs/<id>/manifest.json` and prints the
artifact when ready. **That was the wrong place to fix it.** synapse
is already a real CLI; the wrapper is shadow tooling. The right fix
is at the protocol level.

## Required behavior

Add a `wait` method to the `programs` activation:

```rust
#[plexus_macros::method(streaming, params(
    program_id = "Program id to wait on",
    poll_interval_ms = "How often to check status; default 2000",
    timeout_secs = "Hard cap; default 900 (15 min)"
))]
async fn wait(
    &self,
    program_id: String,
    poll_interval_ms: Option<u64>,
    timeout_secs: Option<u64>,
) -> impl Stream<Item = WaitEvent> + Send + 'static;
```

Events:

```rust
#[derive(Serialize, Deserialize, JsonSchema)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum WaitEvent {
    /// Emitted on status transitions (e.g. NotStarted → Started, Started → Running).
    Progress { status: String, age_ms: u64 },
    /// Emitted once when the program reaches a terminal state.
    Completed { program_id: String, artifact: Value },
    Failed { program_id: String, error: Value },
    /// Emitted if the wait hits the timeout cap.
    TimedOut { program_id: String, last_status: String, waited_ms: u64 },
}
```

Implementation:
1. Reads `programs/<id>/manifest.json` at `poll_interval_ms` cadence.
2. Emits `Progress` only on status changes (don't spam every poll).
3. On `status == "completed"`: read `artifact.json`, emit `Completed`.
4. On `status == "failed"`: read `error.json`, emit `Failed`.
5. On timeout: emit `TimedOut` and end the stream.
6. If the program directory doesn't exist after the first poll cycle:
   emit `Failed` with a "program not found" error.

## Composition

After this lands, the natural synapse usage is:

```bash
# blocking single forecast
synapse -P 4456 substrate forecast update \
    --program-id MY-Q-001 \
    --new-evidence "Will X happen by Y?" \
    --trials 3 \
    --iterative-max-steps 5 \
  | jq -r 'select(.content.type == "started") | .content.program_id' \
  | xargs -I{} synapse -P 4456 substrate programs wait --program-id {}
```

Or expose a helper script that does the pipe (`scripts/forecast.sh`),
but the substrate-side method is the load-bearing piece.

## Acceptance criteria

1. `programs.wait` exists in the programs activation with the shape above.
2. End-to-end live test:
   - Fire `forecast.update` against the running container
   - Pipe its program_id to `programs.wait`
   - The wait stream emits Progress + Completed and renders the artifact natively via synapse's event handling
3. Unit tests cover: completed-quickly, failed, timeout, program-not-found.
4. `cargo test --lib` passes.
5. After this lands:
   - Delete `scripts/mneme` (the Python wrapper)
   - Update README quickstart to use the synapse-pipe pattern
   - The Makefile's `make install` target either becomes a no-op or
     installs only a tiny shell helper (TBD; simpler is better)

## Out of scope

- Listening for program state via WebSocket / push notifications
  (polling is fine; programs are short-lived enough for it).
- Reverse-direction usage where the substrate would push events to a
  client subscription (that's a different ticket).
- Deletion of the marketwatch / ticket-forecast / bench scripts — those
  remain as standalone tools.

## Completion

Method shipped + live verification + scripts/mneme deleted + README
updated to use natural synapse invocations. Status → `Complete`.
