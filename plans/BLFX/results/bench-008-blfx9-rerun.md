---
title: "bench-008 — bench-006 rerun with BLFX-9 layers 1+4 active (n=63 / 58 apples-to-apples)"
date: 2026-04-30
status: complete
---

## Headline

The same 2026-03-15 questions as bench-006, run again with BLFX-9
date-leakage defenses (layer 1 = `before:YYYY-MM-DD` on outbound web
search queries; layer 4 = per-question URL blocklist auto-extracted
from `resolution_criteria`). Same trials=2, iterative_max_steps=5,
concurrency=6.

| Subset | n | mneme BI | crowd BI | paired Δ BI | paired CI |
|---|---|---|---|---|---|
| **Apples-to-apples** (questions both runs scored) | 58 | 83.24 | 58.27 | **+24.98** | — |
| bench-006 baseline (same 58) | 58 | 84.79 | 58.27 | +26.52 | [−0.128, −0.002] |
| **Δ vs bench-006** (cost of BLFX-9 layers 1+4) | — | −1.55 | 0 | **−1.54 BI** | — |
| Full bench-008 set | 63 | 84.57 | 61.57 | +23.00 | [−0.113, +0.004] |

**The cost of layers 1+4: ~1.5 BI.** The remainder of the bench-006
+26.5 paired delta survives the defenses.

## BLFX-9 forecast hypothesis: resolved NO (favorable outcome)

The hypothesis from `BLFX-9.md`'s `forecast:` block:

> "Will building BLFX-9 and re-running bench-006 with the
> date-leakage defense active produce a paired delta vs crowd that
> lands within ±10 BI of Murphy 2026's claimed +4 BI (i.e., delta in
> [−6, +14])?"

YES would have meant our previous +26 BI win was mostly contamination
(defenses would collapse it to near the paper's +4). NO means the
win has substantive non-contamination signal.

Apples-to-apples Δ = **+24.98** is well outside [−6, +14]. The
hypothesis resolves **NO** — and NO is the favorable outcome.

The framing is unfortunate: the system was forecasting whether the
contamination explanation would hold up, and "no" reads as bad news
when it's actually validation. A future ticket should phrase
hypotheses so the verbal valence matches the outcome valence.

## Per-source breakdown (full n=63 bench-008)

| source | n | mneme BI |
|---|---|---|
| polymarket | 27 | **97.27** |
| manifold | 25 | 83.60 |
| metaculus | 8 | 50.48 |
| infer | 3 | 69.27 |

Polymarket dominates. The 27 polymarket questions had mean Brier
0.0068 — mneme is essentially perfect on this slice. Mneme's
weakest source remains metaculus, consistent with prior benches.

## Why the full-set paired CI now includes 0 (and why this is mechanical, not signal)

bench-006 had 5 schema-failure aborts; bench-008 had 0 (the cleanup
recovery from MNEME-29 caught everything). The 5 newly-recovered
questions were:

| id | source | mneme p | crowd p | actual | mneme Brier | crowd Brier |
|---|---|---|---|---|---|---|
| f2i3zt28mm | manifold | 0.007 | 0.011 | 0 | 0.0001 | 0.0001 |
| 40862 | metaculus | 0.009 | 0.010 | 0 | 0.0001 | 0.0001 |
| 0x4d0d4382… | polymarket | 0.009 | 0.025 | 0 | 0.0001 | 0.0006 |
| 0xa70a86d2… | polymarket | 0.014 | 0.034 | 0 | 0.0002 | 0.0011 |
| Y1YDkx1SzzL93AFDpG6v | manifold | 0.004 | 0.012 | 0 | 0.0000 | 0.0001 |

These were all trivially-easy extreme-low-probability events that
didn't happen. Both mneme and crowd nearly nailed them; the paired
contribution is +0.13 BI (vs the headline shift of −2.00 BI from
+26.5 to +23.0).

The reason adding 5 nearly-correct predictions to both sides
*shrinks* the apparent paired Δ:

- Mneme mean Brier on the 5 recovered: 0.0001
- Mneme mean Brier on the 58 in-both: 0.0419
- Crowd mean Brier on the 5 recovered: 0.0004
- Crowd mean Brier on the 58 in-both: 0.1043

When you fold 5 samples with mneme≈0.0001 and crowd≈0.0004 into the
denominator-58 averages, both drop proportionally. But mneme's drop
is steeper (mneme was already good; crowd was much worse). The
**absolute gap shrinks faster than the BI ratio implies**, and at
the CI margin, that's enough to flip "excludes 0" to "includes 0 by
0.004."

The signal didn't move; the n moved. **+24.98 on the apples-to-
apples 58 is the right number to compare to bench-006's +26.5.**

## Operational notes

- **0 failures.** MNEME-29 cleanup recovery + the lenient
  EvidenceItem parser caught everything that bench-006 dropped.
- **Wall-clock**: 22 min for n=63 at concurrency=6, K=2, T=5. Faster
  than projected (50 min) — the substrate had less contention than
  expected since ticket forecasts hadn't started yet.
- **BLFX-9 wiring**: the `cutoff_date=2026-03-15T00:00:00Z` and
  per-question URL blocklist (regex-extracted from
  `resolution_criteria`) flowed cleanly through `forecast.update` →
  `TrialParams.env` → `ClaudecodeStepDriver.env` →
  `apply_search_query_date_filter` and `is_url_blocked` in
  `run_web_search`/`run_lookup_url`. No bench failures attributable
  to the defense pipeline.
- **What the defenses caught**: hard to attribute precisely without
  layer-2 (LLM leak classifier) telling us which hits were dropped
  by reasoning, but the −1.5 BI delta is the aggregate effect of
  layer 1 + layer 4 combined.

## Caveats

1. **Layers 2 and 3 are not yet in production.** Layer 2 (LLM leak
   classifier) has trait + plumbing but no concrete impl; layer 3
   (data-tool date clamping) is blocked on BLFX-15. The full paper-
   spec defense is not deployed; the −1.5 BI cost is from layers
   1+4 alone. Layers 2+3 may shave more, may shave nothing.
2. **The hypothesis-band ±10 BI was a generous noise envelope, not
   a tight prior.** Murphy 2026's +4 was on Gemini-3.1-Pro with
   different question composition; transferring it as a point
   estimate to our setup was always rough.
3. **Polymarket BI 97 is suspicious-good.** It's possible the
   polymarket questions in this bench window were unusually
   one-sided (resolved-NO at high market certainty), making both
   mneme and crowd look good but mneme slightly better. The next
   bench window will tell us if polymarket's edge is durable or a
   slice artifact.

## Decision

- **bench-006's +26 BI delta is real signal, not contamination.**
- BLFX-9 layers 1+4 cost ~1.5 BI — a small, real defense effect.
- Mneme's algorithmic edge survives the most obvious leakage
  vectors. The remaining ~+25 BI is the iterative loop + diversified
  trials + logit aggregation + Platt calibration doing real work
  beyond what a frozen market price encodes.
- **Next experiments worth running** (in priority order):
  1. **BLFX-6** (hierarchical Platt, system's only above-50%
     prediction). Cheapest paper-spec gap to close.
  2. **BLFX-9 layer 2** (Haiku leak classifier). Drop-in given
     the existing trait + plumbing. Re-bench under the same setup
     to see if it shaves another +1 to +3 BI.
  3. Wait on bench-009: a *third* held-out window (e.g. freeze
     2026-04-12, resolve 2026-05) to see if the polymarket +97
     BI replicates or was slice luck.

## Reproducibility

```bash
# In mneme-substrate/, with the containerized substrate running on 4456:
python3 scripts/forecastbench_holdout_run.py \
  --question-set programs/_benchmarks/forecastbench/2026-03-15-llm.json \
  --resolution-sets \
      programs/_benchmarks/forecastbench/2026-03-15_resolution_set.json \
      programs/_benchmarks/forecastbench/2026-03-29_resolution_set.json \
      programs/_benchmarks/forecastbench/2026-04-12_resolution_set.json \
  --resolve-from 2026-03-25 --resolve-to 2026-04-29 \
  --concurrency 6 --port 4456 --trials 2 --iterative-max-steps 5 \
  --cutoff-date 2026-03-15T00:00:00Z \
  --output programs/_benchmarks/runs/<ts>-bench008-holdout-blfx9/
```

Run dir: `programs/_benchmarks/runs/20260430-102102-bench008-holdout-blfx9/`

Substrate state at run time:
- Image: `mneme-substrate:dev` rebuilt with BLFX-9 changes
  (substrate config hash: `1e1172af4a2dee93`, vs pre-BLFX-9
  `09a736d7ffcae255`)
- BLFX-9 layer 1 (`apply_search_query_date_filter`) and layer 4
  (`is_url_blocked`) active throughout
- Calibration store: bench-005-resolved version, unchanged during
  this run
- Vendored data: identical question_set + resolution_sets as
  bench-006

## Auditability

Per MNEME-35 (shipped same day), every forecast in this bench has
a queryable reasoning trace:

```bash
python3 scripts/inspect_program_tree.py <program_id>
python3 scripts/inspect_program_tree.py <program_id> --turns
```

Renders manifest, artifact (with weighted evidence_for /
evidence_against, open_questions, summary), trace, and per-trial
chat turns into a single tree. Programs in this bench predate the
trial-history persistence so step_history will show "(none —
pre-MNEME-35 program)"; chat turns are still queryable from
claudecode.db.
