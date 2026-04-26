---
title: Session Log — autonomous block 2026-04-26
status: handoff
---

# Where the project is when you wake up

## What this session changed

### 1. MNEME-21 — SKILL.md drift bug fixed
Parent claudecode sessions are now content-addressed. The actual session
name is `{logical_name}-skill{8hex(sha256(SKILL.md))}` so that any change
to the prompt produces a fresh session instead of silently reusing a
stale one. `ParentSessionSpec::resolved_name()` is the single source of
truth — every call site (`forecast/activation.rs`, `claudecode_swarm_runtime`)
uses it.

Files: `mneme-substrate/src/mneme/runtime/swarm_runtime.rs:111-122`,
`mneme-substrate/Cargo.toml` (sha2 + hex deps).

### 2. BLFX-2 — verified live end-to-end
Bench `8feb9552` returned a fully populated structured belief state
(probability=0.81, 9 evidence_for, 7 evidence_against, 9 open_questions,
schema 0.2.0) on the S&P-500 question with `trials=2` after the MNEME-21
fix. Took 3m 24s.

Artifact: `mneme-substrate/programs/8feb9552-486a-41d3-bf14-d4d9a310433c/artifact.json`

### 3. BLFX-S03 — token instrumentation (partial)
- `TrialUsage` type added in `mneme-substrate/src/mneme/swarm/mod.rs`
  (mirrors `claudecode::ChatUsage`).
- `TrialResult.usage: Option<TrialUsage>` populated by
  `claudecode_swarm_runtime::run_one_trial` from `ChatEvent::Complete`.
- Per-fan-out totals bubbled into the `swarm_trial` trace entry as
  `usage_total` + `trials_with_usage`.

**Verified live**: bench `e1e52a17` produced
`usage_total = {num_turns: 9, ...}`. `num_turns` works; `input_tokens`,
`output_tokens`, `cost_usd` all come back null because the upstream
claudecode CLI isn't emitting them in its terminal message. See
`mneme/plans/BLFX/results/S03-token-instrumentation.md` for the
diagnosis and follow-up.

### 4. BLFX-4 — iterative loop skeleton (mock-only)
New module `mneme-substrate/src/activations/forecast/iterative_loop.rs`:
- `StepDriver` trait — pluggable per-step LLM driver.
- `iterative_trial` — runs the BLF Algorithm-1 loop: fetch step, parse
  `(action, belief)`, execute action, append to history, repeat until
  `Submit` or `T_max` (force-submit with confidence=Low).
- `QueueStepDriver` — test fixture that dispenses canned responses.
- `build_step_prompt` — deterministic prompt builder; production form
  will move into the SKILL.md eventually.

7/7 tests pass. The claudecode wiring (replacing `run_one_trial` with
the loop using a real claudecode-backed `StepDriver`) is the remaining
work — the load-bearing change for matching the paper.

## Verified state at handoff

- `cargo build --bin mneme-substrate` — green
- `cargo test --lib swarm` — 51/51 pass
- `cargo test --lib iterative_loop` — 7/7 pass
- substrate running on port 4456 (PID via `ps aux | grep mneme-substrate`)
  with the new build that includes all of the above

## Open / not done

- **Token capture is half-instrumented.** `num_turns` works; cost/tokens
  upstream gap. New ticket worth opening: `MNEME-22` "extract
  input/output tokens from claudecode CLI terminal message". Acceptance:
  one-trial bench surfaces non-null `cost_usd` in the trace.
- **BLFX-4 wiring not done.** Skeleton + tests are in. Replacing
  `run_one_trial` with `iterative_trial<ClaudecodeStepDriver>` is the
  next concrete step; estimate ~half a day plus a live bench. Risk: cost
  per trial goes 5–10x (per BLFX-4 plan). Recommend running first live
  loop bench on Sonnet (per S04 result) to amortize.
- **Multi-question micro-benchmark not run.** Original task #26 was a
  3-question bench. Punted in favor of the single-question
  `e1e52a17` verification (which confirmed the new instrumentation).
  When BLFX-4 wires through, run the 3-question bench against both
  iterative-loop and single-shot for an apples-to-apples Brier delta.

## Suggested next moves (in order)

1. Open `MNEME-22` for the upstream token-capture gap (small ticket).
2. Wire `ClaudecodeStepDriver` for `iterative_loop::iterative_trial` —
   this is the BLFX-4 acceptance-criterion work and the single largest
   algorithmic change to match Murphy 2026.
3. Run a 3-question micro-bench comparing single-shot vs iterative on
   Sonnet. Brier delta + cost delta both matter.
4. If iterative wins meaningfully, swap `run_one_trial` over and bump
   `forecast` to v0.3.0.

## Pointers

- All BLFX tickets: `mneme/plans/BLFX/`
- Results so far: `mneme/plans/BLFX/results/`
  - `bench-001-K5-baseline.md` — first K=5 live bench (Sonnet)
  - `S04-base-llm-choice.md` — Sonnet for dev, Opus for benchmarks
  - `S03-token-instrumentation.md` — the partial-instrumentation finding
- Substrate Cargo: `mneme-substrate/Cargo.toml`
- Hub registration: `mneme-substrate/src/builder.rs`
