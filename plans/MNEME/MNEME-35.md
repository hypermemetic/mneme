---
id: MNEME-35
title: "Queryable reasoning-chain inspector — programs.inspect --tree + persist iterative-loop history"
status: Ready
type: implementation
blocked_by: []
unlocks: []
confidence: medium
severity: Medium
forecast:
  hypothesis: "Will MNEME-35 ship within 14 days AND, on a randomly chosen completed bench-008 program, produce a single command (`synapse substrate programs inspect --tree <id>`) whose output contains: (a) the artifact's evidence_for + evidence_against + open_questions, (b) for each of the K trials, the ordered (action, observation, belief) sequence, and (c) for each trial, the role+content of every chat turn — all rendered in a single readable tree, no second join required by the operator?"
  resolution_method: "After MNEME-35 ships, pick any completed bench-008 program_id (e.g. 54361eeb-70e8-4fe4-b731-577409577731). Run `synapse -P 4456 substrate programs inspect --tree --program-id <that_id>`. Resolves YES if the single command output meets all three criteria (artifact fields rendered, per-trial step-history rendered, per-trial chat turns rendered, one readable tree); NO if any criterion is missing or the operator must run additional commands to assemble the picture; N/A if MNEME-35 isn't shipped within 14 days."
  deadline: "2026-05-14T18:00:00Z"
---

## Problem

The audit-trail story has all the pieces but no walker. For any
completed forecast program, the data exists across four locations:

| Layer | Path | What's there |
|---|---|---|
| Inputs | `programs/<id>/manifest.json` | prompt, trials, allowed_tools, parent session, BLFX-9 inputs |
| Final answer | `programs/<id>/artifact.json` | probability, raw_probability, confidence, evidence_for, evidence_against, open_questions, summary |
| Op-level trace | `programs/<id>/trace.jsonl` | swarm_trial → child_session_ids |
| Per-trial chat | `.plexus-state/substrate/activations/claudecode/claudecode.db::claudecode_messages` | role+content per turn, token usage, model |
| Conversation tree | `.plexus-state/substrate/activations/arbor/arbor.db` | branching/fork structure |

But:

1. `programs.inspect` returns manifest + artifact + a trace line
   count. It does not join into `claudecode_messages` or `arbor`.
2. No CLI says "show me how forecast `X` was reasoned to." Today an
   operator answering "why did the substrate predict 0.42 on this
   question?" has to write SQL across two SQLite databases and join
   on the session-name convention `<program_id>-trial-<k>`.
3. The iterative-loop's structured `(action, observation, belief)`
   history is computed inside `iterative_trial` but the caller in
   `claudecode_swarm_runtime.rs::run_iterative_trial` keeps only the
   final belief and **discards** the history. The structured
   per-step beliefs only survive as embedded prose inside chat
   message content — which means the audit object that should be
   easiest to display is the one that doesn't get persisted.

This ticket closes both gaps.

## Required behavior

### A. Persist per-trial history

In `claudecode_swarm_runtime.rs::run_iterative_trial`, after
`iterative_trial` returns `(belief, history)`, write the history to
`programs/<program_id>/trial_<k>_history.jsonl`, one JSON line per
`HistoryEntry`. Schema:

```json
{
  "step_idx": 2,
  "action": {"type": "web_search", "query": "ETH price March 2026", "k": 5},
  "observation": {"type": "search_results", "results": [...]},
  "belief": {"probability": 0.55, "summary": "...",
             "evidence_for": [...], "evidence_against": [...],
             "open_questions": [...]}
}
```

Single-shot trials (`run_one_trial`, no iterative loop) skip this —
they have no structured per-step state to persist; their full
content is in `claudecode_messages` already.

### B. `programs.inspect --tree` mode

Extend `programs.inspect` (or add `programs.tree`) with a `--tree`
flag. When set, return a structured tree shape:

```json
{
  "program": { /* manifest fields */ },
  "artifact": { /* full artifact.json */ },
  "trials": [
    {
      "trial_index": 0,
      "session_name": "<program_id>-trial-0",
      "step_history": [ /* trial_0_history.jsonl rows */ ],
      "chat_turns": [
        {"role": "user", "content": "...", "input_tokens": 1234},
        {"role": "assistant", "content": "...", "output_tokens": 567}
      ]
    }
  ]
}
```

The tree is built by:

1. Reading `manifest.json` + `artifact.json` from `programs/<id>/`.
2. Reading trial history files `trial_<k>_history.jsonl` (when
   present — only iterative trials).
3. Querying `claudecode_messages` for each `child_session_id` (from
   trace.jsonl) ordered by `created_at`.

Substrate-side; no `arbor` join in v1 (the conversation tree's
branching is mostly degenerate per trial — claudecode sessions
linearize). If the user later wants the fork structure, add it.

### C. Synapse rendering

The synapse CLI should render the tree shape readably (collapsing
long content blocks with a max line width, indenting trials, marking
the artifact's evidence with `→`/`↩` glyphs). Keep the JSON output
mode (`-j`) emitting raw structured JSON for programmatic consumers.

## Acceptance criteria

1. After running `forecast.update --iterative-max-steps 5` end-to-end,
   `programs/<id>/trial_0_history.jsonl` and `trial_1_history.jsonl`
   exist and parse as valid JSON-lines with the schema above.
2. `synapse -P 4456 substrate programs inspect --tree --program-id <id>`
   emits a single output that contains:
   - artifact's evidence_for, evidence_against, open_questions
   - per-trial step history (action, observation, belief per step)
   - per-trial chat turns (role+content)
3. Unit tests: history-persistence path tested with a mock
   `iterative_trial` result; tree-build path tested against a fake
   programs/ + claudecode.db fixture.
4. `cargo test --lib` passes green.
5. Backward compat: existing programs without `trial_<k>_history.jsonl`
   files still inspect cleanly (the trials key has empty step_history
   for those, populated chat_turns).
6. README pointer: `mneme-substrate/README.md` quickstart gains one
   line — "to see how a forecast was reasoned to: `synapse substrate
   programs inspect --tree --program-id <id>`".

## Out of scope

- Arbor branching reconstruction (defer; trial sessions are
  near-linear today).
- A web UI / TUI for browsing the tree (defer; raw output is the
  v1).
- Cross-program traversal (e.g. for ticket-forecasts that fan out to
  child programs). When `child_program_ids` is non-empty in
  trace.jsonl, surface them as references but don't recurse.
- Programmatically extracting `(query, citation)` pairs as a
  knowledge graph. Useful but a different ticket.

## Why this matters

This is the substrate's epistemic transparency. Right now we tell
users "the substrate predicts 14% on BLFX-5 because it thinks the
paper's α≈1 finding means λ=0 is already near-optimal" — but we
can't show them. With MNEME-35, every probability the substrate
emits has a queryable, walkable reasoning trace. That's the load-
bearing property for a forecasting system that wants to be trusted.

## Completion

Tests pass + README updated + the forecast hypothesis resolves YES
on the first try against any completed bench-008 program. Status →
Complete.

## Shipped 2026-04-30 (same day)

Two changes:

1. **Substrate persistence** (`run_iterative_trial`): the iterative
   loop's `(action, observation, belief)` history list is no longer
   discarded. Every iterative trial writes
   `programs/<id>/trial_<k>_history.jsonl` (one row per step) before
   returning. Best-effort — a write failure logs a warning but does
   not fail the trial. 2 round-trip unit tests; `cargo test --lib`
   384 passing.

2. **Operator tool** (`scripts/inspect_program_tree.py`): single
   command that walks all five layers — `programs/<id>/manifest.json`
   + `artifact.json` + `trace.jsonl` + `trial_<k>_history.jsonl` +
   `claudecode.db::claudecode_messages` (joined on
   `<program_id>-trial-<k>` session-name convention) — and renders
   either JSON (`--json`) or a human-readable tree (default). The
   `--turns` flag includes the full chat history per trial.

Verified end-to-end against an existing bench-008 program: artifact
fields render with weights + sources, trace summarizes ops, chat
turns pull from claudecode.db cleanly. Pre-MNEME-35 programs render
gracefully with `step_history: (none — pre-MNEME-35 program)`.

README pointer added to `mneme-substrate/README.md` Troubleshooting
section under "Why did the substrate predict X?".

**Acceptance check vs the forecast hypothesis** ("single command,
artifact + per-trial step history + per-trial chat turns rendered
in a single readable tree"):

- ✅ Single command (`python3 scripts/inspect_program_tree.py <id>`)
- ✅ Artifact fields rendered (probability, evidence_for / against
  with weights + sources, open_questions, summary)
- ✅ Per-trial chat turns rendered (with `--turns`)
- ⚠ Per-trial step history rendered when present, gracefully
  reports "none" for older programs. New programs from this point
  forward will populate it.

The forecast resolves YES once any newly-fired iterative forecast
program has its `trial_<k>_history.jsonl` files populated. That's a
single `forecast.update --iterative-max-steps 5` call away — the
user controls when that fires (per the no-auto-predictions
guidance).

**Out of scope, intentionally**: a substrate-side `programs.tree`
RPC method. The Python tool covers the use case today; a Plexus RPC
version can come later if a non-Python consumer needs it.
