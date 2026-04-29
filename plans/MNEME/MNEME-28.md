---
id: MNEME-28
title: "Wire platt_apply into forecast.update artifact construction"
status: Ready
type: implementation
blocked_by: [MNEME-26]
unlocks: [BLFX-6, BLFX-13]
confidence: high
severity: Medium
---

## Problem

`forecast.update`'s background task computes a raw aggregated probability
and writes it directly into the artifact:

```rust
let probability = logit["aggregated"].as_f64().unwrap_or(DEFAULT_PRIOR);
// ... no calibration step ...
let mut state = ForecastState { probability, ... };
```

The pipeline E2E test
(`pipeline_e2e_program_trial_aggregate_calibrate_artifact`) demonstrates
that `platt_apply(bias, raw_aggregated)` is the right call. The
activation should follow that pattern. Without it, the calibration store
fills up (post-MNEME-26) but is never consulted — Platt parameters fit
themselves and then sit unused.

This is the wire-up that turns the calibration loop from "we record
outcomes" into "we record outcomes and the next forecast benefits from
them." It's the prerequisite for BLFX-6 (per-source intercepts) being
measurable.

## Context

- `mneme-substrate/src/mneme/calibration/platt.rs::platt_apply(params,
  p_raw) -> Result<f64, PlattError>` already exists and is tested.
- `mneme-substrate/src/mneme/calibration/store.rs::CalibrationStore::read_bias()
  -> Result<Option<PlattParams>, StoreError>` returns `None` during
  cold-start, `Some(params)` once `record()` has fit them.
- The activation already constructs a `CalibrationStore` indirectly
  through the pipeline test; the production path needs the same
  construction inside `run_update_in_background`. Open it at
  `context.programs_root().join("_calibration")` (matches the test
  convention).
- Cold-start behaviour: `read_bias()` returns `None` → use raw
  aggregated probability unchanged (current behaviour). Only apply
  Platt when `Some(params)` returned.
- Persist both `raw_probability` AND calibrated `probability` in the
  artifact for forensic comparison. The `ForecastState` schema doesn't
  currently have a `raw_probability` field; add one (default to
  Some(p) for backward-compat).

## Required behavior

In `run_update_in_background`:

1. After `let probability = logit["aggregated"]...`, save it as
   `raw_probability`.
2. Open `CalibrationStore::open(context.programs_root().join("_calibration"))`.
3. Call `store.read_bias()`.
4. If `Some(params)`: `let probability = platt_apply(params, raw_probability)?;`
   Else: `let probability = raw_probability;`
5. Set `ForecastState.probability = <calibrated>`,
   `ForecastState.raw_probability = Some(raw_probability)`.
6. If calibration applied, log `tracing::info!("calibration_applied: a={}, b={}, raw={:.4}, calibrated={:.4}", ...)`.

Schema change: add
`pub raw_probability: Option<f64>` to `ForecastState` in
`mneme-substrate/src/activations/forecast/types.rs`. Bump
`BELIEF_SCHEMA_VERSION` to "0.3.0". Update existing v0.2 round-trip
tests so older artifacts (raw_probability=None) still parse.

## Acceptance criteria

1. With empty calibration store: `forecast.update` artifact has
   `probability == raw_probability` (cold-start identity).
2. With ≥10 resolved observations (post-MNEME-26): the artifact's
   `probability` may differ from `raw_probability`. The substrate log
   contains a `calibration_applied` line.
3. `BELIEF_SCHEMA_VERSION` is "0.3.0" and v0.2 artifacts still parse.
4. New unit test in
   `mneme-substrate/src/activations/forecast/activation.rs`:
   "update_applies_platt_when_calibration_store_has_data" — uses the
   `DeterministicMockSwarmRuntime` + a pre-seeded calibration store,
   asserts the post-update artifact has `probability != raw_probability`.
5. `cargo test --lib` passes.

## Out of scope

- Per-source intercepts (BLFX-6 covers that; this ticket is the
  flat-Platt wire-up).
- Re-calibrating in real time as observations land (Platt is fit on
  `record()`; we just consume the latest fit).
- A CLI to manually trigger refit (could be useful but separate).

## Completion

Tests pass; one local sanity-fire of `forecast.update` after MNEME-26
shows the substrate log emitting `calibration_applied`; status →
`Complete`.
