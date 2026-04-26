---
id: MNEME-22
title: "Wire upstream token capture from claudecode CLI Complete event"
status: Ready
type: implementation
blocked_by: []
unlocks: []
confidence: medium
severity: Low
---

## Problem

Substrate-side token instrumentation (BLFX-S03) is in place: `TrialUsage` flows
from `ChatEvent::Complete` through `TrialResult.usage` and is summed into the
`swarm_trial` trace entry. But `input_tokens`, `output_tokens`, and `cost_usd`
come back `None`. Only `num_turns` is populated.

Verified live by the `e1e52a17-469d-4439-aa24-37a40c05fdd2` bench artifact's
trace.jsonl, where `usage_total` looks like:

```json
{"num_turns": 9, "input_tokens": null, "output_tokens": null, "cost_usd": null}
```

The substrate's plumbing is correct (`Option<u64>` preserves `Some` wherever
the SDK provides it). The gap is upstream: the in-process `claudecode`
activation's `Complete` event is constructed without the token fields.

## Context

Two likely root causes (need to investigate):

1. The `claude-code` CLI version in use isn't emitting the `usage` block in
   its terminal `result` message (CLI ABI variability between versions).
2. The substrate's claudecode session loop isn't extracting the `usage`
   block from whichever message field carries it (parser drift).

Code locations:
- `mneme-substrate/src/activations/claudecode/types.rs:268-275` — `ChatUsage`.
- `mneme-substrate/src/activations/claudecode/types.rs:476-482` — `ChatEvent::Complete { usage }`.
- The session-loop code that builds the `Complete` event from CLI output —
  search for `ChatEvent::Complete {` with non-`None` `usage` to find where
  it's *constructed*; the construction site is what needs updating.

## Required behavior

A 1-trial `forecast.update` against a real claudecode-backed substrate produces
a trace entry whose `usage_total` includes non-null `input_tokens`,
`output_tokens`, and `cost_usd`.

## Acceptance criteria

1. Investigation: identify whether the gap is at CLI or parser layer
   (capture an example raw message line in the ticket comments).
2. Fix the construction site for `ChatEvent::Complete` so `usage` is
   populated whenever the CLI emits the data.
3. Live verification: 1-trial `forecast.update` produces a trace where
   `usage_total.input_tokens > 0` and `cost_usd > 0`.
4. `cargo test --lib` passes.

## Out of scope

- Per-event token counting (we want totals at trial close, not deltas
  per Content event).
- Cost dashboards / aggregation over time.
- Anything in `BLFX-S03`'s spike scope beyond the upstream wire.

## Completion

Live verification per (3) above; status flips to `Complete`.
