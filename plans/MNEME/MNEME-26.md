---
id: MNEME-26
title: "Resolve bench-003 predictions into the calibration store"
status: Ready
type: implementation
blocked_by: []
unlocks: [BLFX-5, BLFX-6, MNEME-28]
confidence: high
severity: Low
---

## Problem

`forecast.resolve` (MNEME-23) is wired but the calibration store is empty.
20 predictions from bench-003 (program ids in
`mneme-substrate/programs/_benchmarks/runs/20260426-165255-n20-k2/results.jsonl`)
have known actuals — every question resolved between 2024-12-31 and
2026-04-25. Nothing is feeding them into the store. Until it's seeded with
≥ `COLD_START_THRESHOLD` (10) observations, hierarchical Platt is identity
and LOO-CV α has no data to fit on.

This is a one-time data-import task. ~5 minutes of work. Unblocks BLFX-5
(LOO-CV α) and BLFX-6 (per-source intercepts) as having any non-stub
behavior.

## Context

- `mneme-substrate/programs/_benchmarks/runs/20260426-165255-n20-k2/results.jsonl`
  has one record per resolved question. Each record carries
  `program_id`, `predicted`, `actual`, and the underlying ForecastBench
  question's `resolved_to`.
- `forecast.resolve(program_id, actual: bool, resolved_at?)` writes a
  `ResolvedObservation` to `programs/_calibration/history.jsonl`.
- `actual: bool` requires thresholding the float `resolved_to` at 0.5.
  ForecastBench markets can resolve fractionally on ambiguous outcomes;
  we treat ≥ 0.5 as YES.
- The container's substrate must be running on port 4456 with the
  bind-mounted programs/ for both calibration store + program lookup.

## Required behavior

Write a small idempotent script
(`scripts/resolve_bench_003.py`) that:
1. Reads `programs/_benchmarks/runs/20260426-165255-n20-k2/results.jsonl`.
2. For each record with `program_id` and `actual ∈ {0.0, 1.0}` (skip
   probabilistic resolutions ∈ (0, 1) for now — they need a richer
   `ResolvedObservation` API):
   - Call `synapse -j -P 4456 -p '{"program_id": "<id>", "actual": <bool>,
     "resolved_at": "<resolution_date>T23:59:59Z"}' substrate forecast resolve`.
3. After the loop, inspect `programs/_calibration/history.jsonl`. Confirm
   row count ≥ 10 (or ≥ the number of usable bench-003 records).
4. Print a summary: how many rows added, how many skipped (and why), what
   the post-fit `bias.json` says (Platt should kick in once row count
   crosses `COLD_START_THRESHOLD`).

Idempotent: re-runs should not duplicate. If a record's `program_id`
already appears in `history.jsonl`, skip it.

## Acceptance criteria

1. `scripts/resolve_bench_003.py` exists and runs successfully against
   the running containerized substrate.
2. After running, `programs/_calibration/history.jsonl` has ≥ 15 rows
   (out of 20 bench-003 predictions; some may be skipped if their
   `resolved_to` is fractional).
3. `programs/_calibration/bias.json` exists (Platt fit triggered).
4. Re-running the script is a no-op (idempotent).
5. The script prints which rows were added and which skipped + why.

## Out of scope

- Bulk-resolving from arbitrary other bench runs (separate task; this
  ticket is just the bench-003 backfill).
- Extending `ResolvedObservation` to accept a probabilistic `actual: f64`
  (separate ticket if we need it).
- Re-running bench-003 — we trust the recorded predicted probabilities.

## Completion

Script committed; calibration store has ≥15 rows; status → `Complete`.
