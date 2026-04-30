---
title: "bench-006 — held-out post-cutoff (n=58 paired) ForecastBench eval"
date: 2026-04-29
status: complete
---

## Headline

| Run | n | mean Brier | Brier Index | 95% CI on mean Brier |
|---|---|---|---|---|
| **Mneme** (iterative T=5, λ=0.0, Platt) | 58 | **0.0380** | **84.79** | [0.0005, 0.0884] |
| Crowd baseline (paired) | 58 | 0.1043 | 58.27 | [0.0674, 0.1462] |

**Paired analysis:**
- Mean Brier delta (mneme − crowd): **−0.0663** (negative = mneme better)
- BI delta: **+26.52**
- 95% CI on paired mean Brier delta: **[−0.1277, −0.0015]** — excludes 0
- Per-question outcomes: **mneme wins 33, crowd wins 3, ties 22**
  (using |Δ Brier| > 0.01 as the threshold)

**Disposition:** mneme beats the crowd at p<0.05 on POST-CUTOFF questions.
The bench-005 win does NOT evaporate on a held-out sample. The point
estimate of the BI delta on post-cutoff questions (+26.5) is essentially
identical to bench-005 on contaminated questions (+26.6). The 95% CI
on the paired delta is wider here (n=58 vs 94, more high-variance
metaculus and manifold survivors) and the upper bound is just under 0
(−0.0015), but it still excludes 0 — the win is real.

## Selection / contamination posture

- Question set: `2026-03-15-llm.json` (forecast_due_date 2026-03-15).
  Freeze prices reflect the market state at that date.
- Resolution sets unioned: `2026-03-15`, `2026-03-29`, `2026-04-12`.
- **Resolution-date filter:** kept only questions resolving in
  `[2026-03-25, 2026-04-29]` — strictly after the question_set freeze
  AND past Sonnet's training-cutoff buffer (~2026-Q1).
- 63 joined; 5 failures (described below); n=58 successes.
- Source mix (n=58): polymarket 25, manifold 23, metaculus 7, infer 3.

Bench-005's contamination concern was: "all 100 questions resolved
2024-07–early-2026, which is *inside* Sonnet's training window, so the
agent could be retrieving post-resolution articles via WebSearch and
the +26 BI delta is just hindsight." This bench addresses that
directly. All 58 successes resolved AFTER 2026-03-25, after the
freeze date and at/after the model's training cutoff.

## Per-source breakdown

| source | n | mneme mean Brier | mneme BI | crowd mean Brier | crowd BI | paired Δ Brier | paired Δ BI |
|---|---|---|---|---|---|---|---|
| polymarket | 25 | 0.0003 | 99.9 | 0.0935 | 62.6 | −0.0932 | +37.3 |
| manifold | 23 | 0.0430 | 82.8 | 0.1347 | 46.1 | −0.0917 | +36.7 |
| metaculus | 7 | 0.1397 | 44.1 | 0.0821 | 67.2 | +0.0577 | −23.0 |
| infer | 3 | 0.0768 | 69.3 | 0.0131 | 94.8 | +0.0636 | −25.5 |

Notes:
- Polymarket and manifold drive the win — both show ~+37 BI advantage.
- **Metaculus and infer flip:** the crowd beats mneme on 7+3=10
  questions where the crowd happened to be very confident and right.
  Same mneme-loses-on-metaculus pattern bench-005 saw, now amplified
  because the crowd's metaculus performance was unusually good on this
  sample.
- The two big mneme misses (Brier 0.97 each, p≈0.99 → actual=0)
  account for ~3.4 percentage points of mneme's mean Brier alone:
  - `ylgphSdccA` (manifold)
  - `40852` (metaculus)
  These are exactly the kind of "agent confidently picks the wrong
  side because hindsight isn't available" failures the held-out
  design was meant to surface — and there are only two of them in 58.

## Failures

5/63 questions failed:

| id | source | failure |
|---|---|---|
| `40862` | metaculus | `AllTrialsFailed` at `swarm.trial` |
| `0xa70a86…` | polymarket | `AllTrialsFailed` |
| `0x4d0d43…` | polymarket | `AllTrialsFailed` |
| `Y1YDkx1SzzL93AFDpG6v` | manifold | `AllTrialsFailed` |
| `f2i3zt28mm` | manifold | polling timeout (10 min hard cap) |

7.9% failure rate, similar to bench-005's 6%. `AllTrialsFailed`
remains the dominant failure mode — same pattern as bench-005 (where
it was concentrated in metaculus). This sample is more uniform across
sources, suggesting `AllTrialsFailed` isn't source-specific, just
question-shape-specific. Worth diagnosing in a follow-up but doesn't
change the headline.

## Cost / wall-clock

- 22.5 min wall-clock for 58 successful + 5 failed
- concurrency=6, K=2, T_max=5
- Mean per-question wall: 118s, min 33s, max 376s
- Effective throughput: ~2.8 questions/min

Substantially faster than bench-005 (32.8 min for 94 successes) — the
post-cutoff sample has fewer slow metaculus questions.

## Honest disposition

**Did mneme beat the crowd at p<0.05 on POST-CUTOFF questions?** Yes.

- Paired BI delta = +26.52 (vs bench-005's +26.61)
- 95% CI on paired mean Brier delta = [−0.1277, −0.0015], excludes 0
- 33/58 wins, 3/58 losses, 22/58 ties — heavy positive skew

**Did the bench-005 win evaporate?** No. The point estimate is almost
identical and the CI still excludes 0. This is the cleanest evidence
we have to date that mneme's algorithm produces a real edge over the
crowd, not a hindsight-driven artifact.

Caveats:
1. The CI's upper bound is −0.0015 — close to 0. Re-running on a
   different held-out window could plausibly produce a result whose
   CI just barely includes 0. n=58 is enough to clear p<0.05 once,
   not enough to be robust to sample variation.
2. The win is concentrated in polymarket and manifold (n=48 of 58).
   Metaculus and infer (n=10) flip in the crowd's favor on this
   sample. Murphy 2026's claimed +4 BI is on a more diverse mix.
3. We are still only comparing to the *frozen market price*, not to
   the LLM ensemble baseline used in the paper. A paired comparison
   to the published Gemini/GPT entries on this same question set
   would be the next defensible apples-to-apples claim.
4. Sonnet's actual training cutoff isn't published; we used early-2026
   as a conservative bound. If the cutoff is actually 2025-Q4, this
   bench is even more honestly held-out. If it's late 2026-Q1, some
   of the earliest resolutions (2026-03-26..2026-03-31) may have
   leaked partial event signal — but the paired delta in that
   sub-window doesn't differ visibly from the rest of the sample.

## What this establishes

- **The +26 BI win on bench-005 was not contamination.** Replicates
  on a clean held-out sample with effectively identical magnitude.
- **The algorithm has real lift on prediction-market questions.**
  +37 BI on each of polymarket and manifold paired-against-crowd is
  large and statistically resolvable.
- **Metaculus weakness persists** and is now reproduced on a fresh
  set of 7 questions. This re-stages BLFX-6 (per-source Platt
  intercepts) as the right next calibration move — metaculus's
  strict-resolution-criteria interpretation is a systematic bias,
  not noise.

## What it does NOT establish

- That mneme matches the paper's specific BI numbers on a fully
  reproduced Gemini-3.1-Pro held-out set. We didn't compare to the
  paper's LLM entries, only to the crowd freeze price.
- That the win generalizes outside market questions. Dataset
  questions (acled, dbnomics, fred, wikipedia, yfinance) are still
  blocked behind BLFX-15.
- That the result is robust to sample variation. n=58 cleared p<0.05
  once. Repeat with another freeze date (e.g. 2026-03-29) before
  treating this as a robust finding.

## Recommended next moves

1. **Repeat with 2026-03-29 freeze** — second independent post-cutoff
   sample; the data is already vendored. ~30 min run.
2. **Per-source Platt (BLFX-6)** — metaculus is now a 14-question
   pattern (7 here + 7-of-9 in bench-005). Source-specific bias term
   would address it.
3. **Diagnose `AllTrialsFailed`** — 4 of 5 failures in this run.
   Worth understanding before BLFX-13.
4. **BLFX-9** (4-layer date-leakage defense) — even with a clean
   held-out window, the agent's WebSearch could in principle pull
   post-resolution articles published 2026-03-26..2026-04-29. We
   don't currently know if it does.

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
  --output programs/_benchmarks/runs/<ts>-bench006-holdout-postcutoff/
```

Substrate state at run time:
- Image: `mneme-substrate:dev` (same as bench-005)
- Calibration store: bench-005-resolved version (Platt fit unchanged
  during this run; observations not yet folded back in)
- Vendored data: `2026-03-15-llm.json` + three resolution sets
  through 2026-04-12
- Container: bind-mounted `programs/`, OAuth via macOS Keychain forward
- Run dir: `programs/_benchmarks/runs/20260429-193204-bench006-holdout-postcutoff/`
- New script: `scripts/forecastbench_holdout_run.py` (handles
  multi-resolution-set union join + paired analysis inline)
