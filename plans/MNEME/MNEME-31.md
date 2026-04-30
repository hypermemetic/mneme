---
id: MNEME-31
title: "Tickets-as-forecasts: every design decision becomes a recorded prediction"
status: Ready
type: implementation
blocked_by: []
unlocks: []
confidence: medium
severity: Medium
forecast:
  hypothesis: "Will MNEME-31's Phase-1 script ship within 24 hours and produce ≥3 recorded design-forecasts (BLFX-9 prediction + 2 others) by 2026-05-01?"
  resolution_method: "Check `plans/_predictions.jsonl` exists and has ≥3 entries; check `scripts/ticket_forecast.py` exists and is invocable."
  deadline: "2026-05-01T00:00:00Z"
---

## Problem

We have ~40 well-formed tickets and three benches' worth of evidence
about what the system does. We do **not** have any record of the
*predictions* we made when we filed those tickets. "I think BLFX-4
will help" became "BLFX-4 helped by +15 BI" without anyone ever
recording what they expected up front. This is the calibration vacuum
for design judgment specifically.

The maximalist fix: every ticket carries a forecast in its frontmatter.
At ticket creation, the substrate fires `forecast.update` against the
ticket's hypothesis. The prediction is recorded. When the ticket is
closed, `forecast.resolve` fires with the actual outcome. The
calibration store grows with **predictions about the system's own
design**, separate from event forecasts.

After 30-50 such resolved design-forecasts, we have actual data on
whether mneme's design intuitions are trustworthy — meta-evidence
about the predictor as actor.

## Context

- The ticket format already exists in `plans/<EPIC>/<ID>.md` files
  with YAML frontmatter (id, title, status, blocked_by, unlocks,
  confidence, severity).
- forecast.update / forecast.resolve are wired and proven (MNEME-23,
  MNEME-26, MNEME-28).
- The discipline of "ticket body is a prompt for the executing
  cognate" (per earlier conversation) is in place — bodies already
  contain Problem / Context / Required behavior / Acceptance.
- This ticket has its OWN `forecast:` block in its frontmatter as
  a self-bootstrapping example.

## The new frontmatter field

```yaml
forecast:
  hypothesis: "Will <outcome> by <date>?"
  resolution_method: "Test command, file path to check, or natural-language resolver"
  deadline: "ISO 8601 datetime"
  # Optional:
  threshold: "specific bool predicate when resolution_method outputs a number"
  prior: 0.5  # caller's expected probability if they want to record it
```

Resolution methods can be:
- A shell command whose exit code or output decides (e.g.,
  `cargo test --lib X` → exit 0 = YES)
- A predicate against a known result file (e.g., `bench-013 BI delta
  within ±5` requires reading the result doc)
- Natural language ("Did the change land in main by deadline?")
  which a human or a future agent resolves

## Required behavior

**Phase 1 (this ticket): a script.**

`scripts/ticket_forecast.py` does:

1. Walks `mneme/plans/<EPIC>/*.md`, parses each ticket's YAML
   frontmatter. Skips tickets without a `forecast:` block.
2. For each predicted ticket not yet in `plans/_predictions.jsonl`:
   - Constructs the prompt: ticket body + frontmatter forecast spec
   - Fires `forecast.update` via synapse
   - Records the prediction in `plans/_predictions.jsonl`:
     ```json
     {
       "ts": "2026-04-30T...",
       "ticket_id": "MNEME-31",
       "program_id": "<from substrate>",
       "hypothesis": "Will MNEME-31's Phase-1 script ship...",
       "deadline": "2026-05-01T...",
       "resolution_method": "...",
       "predicted_p": 0.78,
       "raw_predicted_p": 0.81,
       "summary": "..."
     }
     ```
3. Idempotent — re-running skips already-predicted tickets.

`scripts/ticket_resolve.py`:
1. Reads `plans/_predictions.jsonl`.
2. For each prediction past its deadline, prompts the user (CLI) to
   record YES / NO / SKIP / N/A.
3. On YES/NO: calls `forecast.resolve` against substrate so the
   calibration store grows.
4. Records the resolution in `plans/_resolutions.jsonl`.

**Phase 2 (follow-up ticket): a substrate-side `tickets` activation.**

- `tickets.upload(path | content)` parses + validates a ticket file
- `tickets.list(filter)` queries by status, type, project, etc.
- `tickets.predict(ticket_id)` fires the forecast (replaces Phase 1
  script)
- `tickets.resolve(ticket_id, actual)` closes the loop
- `tickets.compile(epic_id)` emits a Lattice graph from a set of
  Ready tickets (the orcha integration sketched in earlier
  conversation)
- Filesystem-as-truth, SQLite-as-index (same pattern as Programs)

Phase 2 is a real chunk of work (~2 days). Phase 1 demonstrates the
pattern at <1 day cost.

## Acceptance criteria

**Phase 1 (this ticket):**

1. `scripts/ticket_forecast.py` exists and runs end-to-end against
   the live substrate.
2. `scripts/ticket_resolve.py` exists with a CLI prompt UX.
3. After one run, `plans/_predictions.jsonl` has ≥3 entries
   (this ticket's prediction + at least 2 others — BLFX-9 and one
   more should also have `forecast:` blocks added retroactively).
4. The recorded predictions show non-degenerate probabilities
   (i.e. not all 0.5; the system actually engaged with the design
   hypotheses).
5. README updated to document the pattern.

**Phase 2:** filed as MNEME-32 follow-up, not required for this
ticket's completion.

## Out of scope

- Phase 2 substrate-side activation (separate ticket)
- Multi-resolver architecture (start with manual CLI prompt for
  resolution; auto-resolution via test commands is Phase 2.5)
- Rich dashboards / queries over predictions (jq is enough)

## Completion

Scripts shipped + ≥3 design-forecasts recorded + README updated +
this ticket's own forecast resolved (whether successfully or not).
Status → `Complete` (the prediction may resolve YES or NO; either
way the ticket's done).
