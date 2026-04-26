---
title: "BLFX-S03 — Token instrumentation findings"
date: 2026-04-26
status: partial-success
---

## What was wired

1. `TrialUsage { input_tokens, output_tokens, cost_usd, num_turns }` added to
   `mneme-substrate/src/mneme/swarm/mod.rs`. Mirrors `claudecode::ChatUsage`
   field-for-field but lives in the swarm namespace so swarm types don't
   depend on claudecode types.
2. `TrialResult.usage: Option<TrialUsage>` (default `None`). Captured by
   `claudecode_swarm_runtime::run_one_trial` from the `ChatEvent::Complete`
   event's `usage` field.
3. `TrialUsage::merge` for field-wise summation across trials.
4. `claudecode_swarm_runtime::trial` sums per-trial usage and emits
   `usage_total` + `trials_with_usage` into the `swarm_trial` trace entry's
   `args_summary`.

## What works

Single-trial bench (`programs/e1e52a17-469d-4439-aa24-37a40c05fdd2/`):

```json
{
  "op": "swarm_trial",
  "args_summary": {
    "n": 1,
    "trials_with_usage": 1,
    "usage_total": {
      "num_turns": 9,
      "input_tokens": null,
      "output_tokens": null,
      "cost_usd": null
    }
  },
  "duration_ms": 157156
}
```

`num_turns: 9` confirms the SDK's terminal Complete event is firing and
`run_one_trial` is correctly capturing `usage` and bubbling it through
`TrialResult` → trace.

## What's missing

`input_tokens`, `output_tokens`, and `cost_usd` come back `null`. The
substrate-side instrumentation is correct (the `Option<u64>` plumbing
preserves `Some` wherever the SDK provides it), so the gap is upstream:
the in-process claudecode activation's `Complete` event isn't getting
token/cost values from the underlying `claude-code` CLI process.

Two likely causes:
- The CLI version in use isn't emitting the `usage` block in its terminal
  message (CLI ABI variability).
- The substrate's claudecode loop isn't extracting the `usage` block from
  whichever message field carries it (parser drift).

## Follow-up

- **Short-term**: trust `num_turns` as a cost proxy until tokens are wired
  (BLFX-2 baseline showed ~K=5 trials → ~$0.23, so per-turn cost is
  bounded; rough `num_turns × $0.005` is a safe upper estimate on Sonnet).
- **Ticket**: open MNEME-22 (or fold into MNEME-20 Phase B/D follow-up) to
  trace where the upstream `claude-code` CLI emits token counts and wire
  the parser end-to-end. Acceptance: a 1-trial WebSearch bench surfaces
  non-null `input_tokens` and `cost_usd` in the trace.

## Build / test status

- `cargo build --bin mneme-substrate` — green
- `cargo test --lib swarm` — 51/51 pass
- `cargo test --lib iterative_loop` — 7/7 pass (new BLFX-4 skeleton)
