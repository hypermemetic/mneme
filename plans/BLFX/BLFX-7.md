---
id: BLFX-7
title: "Source-specific empirical priors π_q"
status: Ready
type: implementation
blocked_by: []
unlocks: [BLFX-13]
confidence: high
---

## Problem

The paper provides each question with a source-specific empirical prior `π_q` — the historical base rate for that question source and subtype. This anchors LLM reasoning when the source has a strong prior (e.g., "yes" base rates on Polymarket vs dataset questions). Paper says: *"The empirical prior has negligible effect on BLF (which acquires better data via search), but helps the no-LLM baseline."* — so it's mainly a baseline-strengthening feature, but it's still part of the standard input the model sees.

## Context

For each question source + subtype, we need a base rate. The paper's Table H.2 (in section §C.7, referenced from §3) lists these. They're empirical rates computed from training data per source.

## Evidence

This ticket is `high` confidence because it's mostly a lookup table + one prompt-template field. The work is collecting the priors from the paper or from training data, storing them, and threading them into the prompt. Implementation is a few-hour task once the priors are tabulated.

## Required behavior

```rust
pub struct EmpiricalPriors {
    inner: HashMap<(String, String), f64>,    // (source, subtype) → base rate
}

impl EmpiricalPriors {
    pub fn load_from_file(path: &Path) -> Result<Self, Error>;
    pub fn lookup(&self, source: &str, subtype: &str) -> Option<f64>;
    pub fn lookup_or_default(&self, source: &str, subtype: &str) -> f64; // 0.5 default
}
```

`forecast.update` accepts an optional `source: Option<String>` and `subtype: Option<String>` parameter. If both provided, the empirical prior is looked up and threaded into the prompt as: *"Empirical base rate for this source and subtype: π_q = X.XX"*.

| Scenario | Outcome |
|----------|---------|
| Source + subtype provided, prior exists | Prior injected into prompt template |
| Source + subtype provided, prior missing | Default 0.5 used; logged warning |
| Source/subtype not provided | No prior in prompt; trial relies on its own reasoning |

## Risks

| Risk | Mitigation |
|------|-----------|
| Paper's exact priors not available | Compute approximations from public data (Polymarket history, Manifold history); document the approximation |
| Subtype taxonomy is paper-specific | Define our own taxonomy; document the mapping |
| Prior staleness (priors drift over time) | Re-fit annually; store fit date in the file |

## What must NOT change

- The forecast.update method signature for callers who don't pass source/subtype (they get the current behavior).

## Acceptance criteria

1. `mneme-substrate/data/empirical_priors.json` (or similar) with priors for at least: polymarket-binary, manifold-binary, metaculus-binary, dataset-binary.
2. `EmpiricalPriors` loader + lookup in `mneme/calibration/priors.rs`.
3. `forecast.update` accepts optional `source` + `subtype` params.
4. Prompt template includes the prior when both params provided.
5. Unit tests: load file, lookup hit, lookup miss, lookup_or_default fallback.
6. `cargo test --lib` passes green.

## Completion

Tests pass + live test with `--source polymarket --subtype binary` shows the prior in the program's manifest inputs and the prompt history; status flips to `Complete`.
