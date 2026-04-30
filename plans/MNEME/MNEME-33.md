---
id: MNEME-33
title: "Resilience for periodic downtime: orphan-forecast recovery + calibration-feed retry + market-deletion handling"
status: Ready
type: implementation
blocked_by: []
unlocks: []
confidence: high
severity: Medium
forecast:
  hypothesis: "Will MNEME-33 ship within 24 hours and survive a simulated downtime cycle (kill substrate mid-marketwatch, restart, recover; resolve sweeper retries failed calibration feeds; deleted Manifold markets handled gracefully) without losing a single pairing or skipping a recoverable resolution?"
  resolution_method: "Run scripts/marketwatch_live.py until it has 5 pairings. Kill substrate mid-second-run. Restart. Re-run marketwatch_live.py — verify (a) no duplicate pairings, (b) no orphan substrate programs without pairings, (c) on simulated substrate-down + retry, the calibration store eventually gains the missing row. Resolves YES if the integration test in scripts/test_marketwatch_resilience.sh passes; NO if any failure mode loses data; N/A if MNEME-33 isn't built within deadline."
  deadline: "2026-05-01T18:00:00Z"
---

## Problem

The marketwatch pipeline currently assumes the substrate is up whenever
the scripts run. That breaks when the substrate is down (laptop closed,
container stopped, host migration), when Manifold markets get
N/A'd/deleted, or when `marketwatch_live.py` is killed between
`fire_forecast()` returning and `append_pairing()` writing.

Three failure modes:

1. **Orphan forecasts.** `marketwatch_live.py` fires a forecast,
   substrate completes it (artifact written to `programs/<id>/`), but
   the script crashes before appending to `pairings.jsonl`. We lose
   the pair record; the artifact lives on but is unreachable from the
   marketwatch view.

2. **Failed calibration feeds.** `marketwatch_resolve.py` finds a
   resolved Manifold market, appends to `resolutions.jsonl`, and tries
   `forecast.resolve` to grow the calibration store. If the substrate
   is down, the synapse call fails. The resolution is recorded but
   `fed_calibration: false` — and on the next sweep,
   `already_recorded_market_ids()` includes this market_id, so we
   never retry. The calibration store silently misses a row.

3. **Deleted/N/A Manifold markets.** A market we forecasted gets
   deleted, cancelled, or resolved as `MKT` (proportional). Current
   code handles `MKT` by recording without feeding calibration — but
   doesn't handle 404s from Manifold gracefully (the resolve script
   would crash mid-loop and leave subsequent markets unswept).

All three are real over a multi-week deployment.

## Required behavior

**1. Orphan-forecast recovery on `marketwatch_live.py` startup.**

Before firing new forecasts, scan `programs/` for `MANIFOLD-LIVE-<market_id>`
program directories whose `manifold_<market_id>` doesn't appear in
`pairings.jsonl`. For each orphan: read the artifact, append a
reconstructed pairing row with a `recovered: true` field. Idempotent.

If the artifact is missing (program failed mid-flight), record a
failure pairing.

**2. Two-phase resolution recording in `marketwatch_resolve.py`.**

Split current single-pass into two phases:

- **Phase A (network-only)**: hit Manifold for each unresolved market,
  write the observed resolution to `resolutions.jsonl` immediately.
  Don't touch substrate yet. Field: `fed_calibration: pending`.
- **Phase B (substrate-only)**: read `resolutions.jsonl`; for each row
  with `fed_calibration in {pending, failed}` AND `resolution in {YES, NO}`,
  call `forecast.resolve` for each forecast we made on that market;
  update the row's `fed_calibration` to `done` or `failed: <error>`.

Idempotent at row level: re-running phase B retries failed feeds
without re-firing successful ones. We need to mutate `resolutions.jsonl`
in place (rewrite atomically) for this; today it's append-only.

Acceptable simplification: keep `resolutions.jsonl` append-only and add
a separate `_calibration_feeds.jsonl` that records (market_id, status,
ts). On retry, look up the most recent status per market_id.

**3. Manifold 404/deletion handling.**

In `marketwatch_resolve.py::fetch_market`: on HTTP error or empty body,
log + treat the market as "still open" (skip resolution, don't crash).
The user can manually purge truly-dead markets if needed.

In `marketwatch_live.py`: if a watched market is gone, the price-delta
check would fail. Skip + log + don't refresh.

**4. Substrate-availability check.**

At the top of both scripts: try a no-op synapse call (e.g.
`synapse -P 4456 substrate _info`); if it fails, exit with a clear
"substrate at port 4456 not reachable; bring it up with `make run` first"
message. For `marketwatch_resolve.py` Phase A specifically, the
substrate isn't strictly needed (Phase A is just Manifold + filesystem);
allow `--phase a-only` to run resolution observation without substrate.

**5. Integration test script.**

`scripts/test_marketwatch_resilience.sh` — orchestrates the failure
scenarios:

- Run marketwatch_live with --max-markets 3
- Verify 3 pairings recorded
- Kill substrate mid-second marketwatch_live invocation; restart
- Re-run marketwatch_live; verify orphan recovered
- Stop substrate; run marketwatch_resolve; verify Phase A wrote to
  resolutions.jsonl with `fed_calibration: pending`
- Restart substrate; re-run marketwatch_resolve; verify retry succeeds
- Verify calibration store rowcount increased

## Acceptance criteria

1. Three changed files: `scripts/marketwatch_live.py`,
   `scripts/marketwatch_resolve.py`, `scripts/test_marketwatch_resilience.sh`.
2. The integration test passes end-to-end against a running container.
3. Manual scenario: stop substrate via `make down`, run `marketwatch_resolve.py`
   — Phase A still works, exits cleanly. Bring substrate back up; re-run
   — Phase B retries the pending feeds.
4. No new Rust changes (this ticket is pure Python operator scripts).
5. README's marketwatch_README.md updated to document the
   up/down lifecycle and the integration test.

## Out of scope

- Full distributed-systems durability (no need for two-phase commit;
  simple retry-on-restart is sufficient for the use case).
- Watching state on a separate process / daemon (cron is fine).
- Alerting on prolonged substrate-down state.

## Completion

Tests pass + integration test green + README updated. Status →
`Complete`. The first long-deployment window (week+ of cron-driven
operation) becomes safe to run.
