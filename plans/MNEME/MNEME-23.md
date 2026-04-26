---
id: MNEME-23
title: "forecast.resolve: record actual outcomes so the calibration store can grow"
status: Ready
type: implementation
blocked_by: []
unlocks: [BLFX-6]
confidence: high
severity: High
---

## Problem

`forecast.update` writes program artifacts containing predicted probabilities,
but there's no method to record the actual resolved outcome of a forecast.
The `CalibrationStore` (`mneme-substrate/src/mneme/calibration/`) has a
`record(ResolvedObservation)` API but no caller — it stays cold-start forever.

Without resolution recording:
- Hierarchical Platt calibration (BLFX-6) has no data to fit on.
- The "calibration" applied at aggregation time is identity (cold-start branch).
- We can't measure our own Brier index over time.
- The whole calibration loop the paper describes is an open-loop system.

## Context

The substrate already has the storage shape:
- `mneme-substrate/src/mneme/calibration/mod.rs::CalibrationStore`
- `ResolvedObservation { program_id, predicted, actual: bool, deadline, resolved_at }`
- `COLD_START_THRESHOLD` gates Platt fitting until enough observations accumulate.

What's missing is the *Plexus method* to record observations and the
substrate-side wiring that maps a `program_id` → its predicted probability
→ a new `ResolvedObservation` row.

## Required behavior

New method on the `forecast` activation:

```rust
#[plexus_macros::method(streaming)]
async fn resolve(
    &self,
    program_id: String,
    actual: bool,
    resolved_at: Option<DateTime<Utc>>,  // defaults to now
) -> impl Stream<Item = ResolveEvent>;
```

Where `ResolveEvent` is:

```rust
pub enum ResolveEvent {
    Recorded {
        program_id: String,
        predicted: f64,
        actual: bool,
        n_observations_total: u64,
    },
    Error { message: String },
}
```

Behavior:
1. Look up the program's manifest + artifact at `programs/<id>/`.
2. Validate: status == Completed, artifact has a `probability` field.
3. Construct a `ResolvedObservation` from the artifact's predicted
   probability and the user-supplied `actual`.
4. Append to the calibration store.
5. Emit `Recorded` with the new total observation count.

## Why now

This is a small ticket (~half a day) but unlocks BLFX-6 (hierarchical Platt)
which is otherwise impossible to validate. Without resolution recording we
have no path to measure whether our forecasts are even reasonable, let alone
calibrated.

## Acceptance criteria

1. `forecast.resolve` Plexus method exists with the shape above.
2. Calls append to the existing `CalibrationStore`.
3. Synapse can call it: `synapse -P 4456 -p '{"program_id":"...","actual":true}' substrate forecast resolve`.
4. Unit test with a mock context proves: open program → close as completed
   with artifact → call resolve → store grows by 1 → predicted matches the
   artifact's probability.
5. Live test: resolve one of the bench programs from this session
   (`8feb9552`, `2c8f52f2`, or `e1e52a17`) and observe the calibration
   store's count incrementing.

## Out of scope

- UI for browsing resolved/unresolved programs (programs activation already
  lists; future work).
- Auto-resolution from prediction-market settlement (much later).
- Bulk-import of historical resolutions (separate ticket if needed).

## Completion

Tests pass + live verification per (5); status flips to `Complete`.
