---
title: "bench-005 — paired n=100 with paper-aligned config"
date: 2026-04-29
status: complete
---

## Headline

| Run | n | mean Brier | Brier Index | 95% CI on mean Brier |
|---|---|---|---|---|
| **Mneme** (iterative T=5, λ=0.0, Platt) | 94 | **0.0398** | **84.07** | [0.0161, 0.0710] |
| Crowd baseline (paired) | 94 | 0.1064 | 57.46 | [0.0746, 0.1438] |

**Paired analysis:**
- Mean Brier delta (mneme − crowd): **−0.0665** (negative = mneme better)
- BI delta: **+26.61**
- 95% CI on paired mean Brier delta: **[−0.1047, −0.0284]** — excludes 0
- Per-question outcomes: **mneme wins 41, crowd wins 8, ties 45**
  (using |Δ Brier| > 0.01 as the threshold)

**Decision per BLFX-18 pre-registered rule:** mneme decisively beats the
crowd at p<0.05. Default `iterative_max_steps` flipped from None to Some(5).

## Per-source breakdown

| source | n | mean Brier | BI |
|---|---|---|---|
| manifold | 25 | 0.0160 | 93.6 |
| polymarket | 57 | 0.0406 | 83.7 |
| infer | 3 | 0.0429 | 82.9 |
| metaculus | 9 | 0.0999 | 60.0 |

Metaculus questions remain the hardest source for mneme — same pattern
seen in bench-003/004 (the Russian-Olympics question 15796 still
mis-resolves). This pre-stages BLFX-6 (per-source Platt intercepts) as
the next high-value calibration improvement; metaculus's pattern of
strict-resolution-criteria interpretation could be corrected with a
source-specific bias term.

## Failures

6/100 questions failed:

| id | source | failure |
|---|---|---|
| xv86CDBe0flxF2epvO3f | manifold | polling timeout (10 min hard cap) |
| 17178 | metaculus | program error (AllTrialsFailed at swarm.trial) |
| 17179 | metaculus | program error |
| 15619 | metaculus | program error |
| 0xc5db10... | polymarket | program error |
| 15010 | metaculus | program error |

5/6 are metaculus questions — likely the iterative agent's response
parser hit format-violation issues mid-loop on those question shapes.
Worth diagnosing in a follow-up but doesn't change the headline. xv86
is the same Olympics-AI question that failed in bench-004 with a 200s
runtime; bumping its hard cap or capping iterations at 3-4 instead of
5 would likely save it.

## Cost / wall-clock

- 32.8 min wall-clock for 94 successful + 6 failed
- concurrency=6, K=2, T_max=5
- Effective throughput: ~3 questions/min

For a full 188-question run at the same config: ~65 min projected.
For a 500-question run (a real BLFX-13 sample): ~3 hours.

## Caveats (the part to read carefully)

This is the same caveat from bench-004, restated more strongly because
the n=100 result amplifies it:

1. **Test-set contamination is severe.** All 100 questions resolved
   between 2024-07 and early 2026, well within Sonnet's training
   knowledge cutoff. The web is full of articles that explicitly
   discuss how each market resolved. Mneme's iterative loop +
   WebSearch retrieves those articles. **The +26.6 BI delta over the
   crowd is largely the gap between "agent with after-the-fact
   knowledge" and "market frozen before resolution."** This is NOT
   the same as Murphy 2026's claimed +4 BI on questions Gemini-3.1-Pro
   plausibly hadn't seen.
2. **n=94 is good enough to be statistically decisive on this sample
   but says nothing about generalization.** A held-out evaluation on
   questions resolving in 2026-Q3+ (post-cutoff) is what would produce
   defensible numbers.
3. **6/100 failures, 5 of which are metaculus.** A meaningful fraction
   of the dataset's hardest source is silently dropping. Worth
   diagnosing before BLFX-13 (the integrated paper-comparison run).

## What the n=100 result establishes

- The infrastructure works at scale (32 min, 6% failure rate, all
  successes parse cleanly).
- The paper-aligned algorithm (λ=0.0 + Platt + iterative T=5) produces
  decisive wins over a non-trivial baseline (the crowd) on contaminated
  questions. **This is a positive control:** the algorithm CAN beat
  smart baselines when given an information advantage. If we'd seen no
  delta here, the algorithm would be broken.
- We have first organizational calibration data: ~120 resolved
  observations (20 from bench-003 + ~94 from bench-005), enough for
  Platt to refit with new bias estimates.

## What it DOES NOT establish

- That mneme would beat the crowd on truly unseen questions. We do
  not know.
- That mneme matches Murphy 2026's specific BI numbers. The paper's
  numbers are on Gemini-3.1-Pro on a held-out sample we don't fully
  reproduce.
- That metaculus-source questions work at all reliably. 5/9 metaculus
  questions failed; the 9 successes had BI 60 (worst by source).

## Decision: defaults flipped

Substrate now defaults `iterative_max_steps` to `Some(5)` when callers
pass `None`. `Some(0)` is preserved as an explicit opt-out for
single-shot mode (regression testing or cost optimization).

`DEFAULT_ITERATIVE_MAX_STEPS = 5` constant added with a comment
referencing this bench's result.

## Recommended next moves

In rough order of value:

1. **Resolve bench-005's predictions into the calibration store** —
   another ~94 rows on top of the existing 20. The Platt fit will get
   meaningfully tighter.
2. **Diagnose the 6 failures** — particularly the 5 metaculus ones.
   Pattern suggests the iterative loop's per-step JSON contract has a
   blind spot on certain metaculus question phrasings.
3. **BLFX-9** (4-layer date-leakage defense). After this lands we can
   honestly claim our numbers reflect algorithm quality, not lookup
   advantage.
4. **BLFX-13** (full integrated benchmark + ablation sweep) — the
   final paper-comparison report.
5. **Start logging real forward-looking forecasts** outside the
   benchmark. A small set of questions resolving in 2026-Q3-Q4 used
   as a manually-curated held-out test, resolved over time.

## Reproducibility

```bash
# In mneme-substrate/, with the containerized substrate running:
python3 scripts/forecastbench_live_run.py \
  --question-set programs/_benchmarks/forecastbench/2024-07-21-llm.json \
  --resolution-set programs/_benchmarks/forecastbench/2024-07-21_resolution_set.json \
  --n 100 --concurrency 6 --port 4456 --trials 2 --iterative-max-steps 5 \
  --output programs/_benchmarks/runs/<ts>-bench005/
```

Substrate state at run time:
- Image: mneme-substrate:dev (built from MNEME-28 commit)
- Calibration store: 20 obs from bench-003, Platt a=0.5947 b=−0.3998
- Vendored data: 2024-07-21 question/resolution release
- Container: bind-mounted programs/, OAuth via macOS Keychain forward
- Run dir: `programs/_benchmarks/runs/20260429-131700-bench005-n100-iterative/`
