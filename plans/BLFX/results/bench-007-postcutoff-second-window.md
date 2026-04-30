---
title: "bench-007 — second held-out post-cutoff window (2026-03-29 freeze)"
date: 2026-04-29
status: complete
---

## Headline

| | n | mean Brier | BI | 95% CI mean Brier |
|---|---|---|---|---|
| **Mneme** (iterative T=5, λ=0.0, Platt, lenient parser, cleanup recovery) | 27 | **0.0714** | **71.43** | [0.0057, 0.1629] |
| Crowd (paired, freeze 2026-03-29) | 27 | 0.2226 | 10.98 | [0.1485, 0.3093] |

**Paired analysis:**
- Mean Brier delta (mneme − crowd): **−0.1511**
- **BI delta: +60.45**
- 95% CI on paired mean Brier delta: **[−0.2684, −0.0239]** — excludes 0
- Per-question: **mneme wins 21, crowd wins 3, ties 3**

**Per-source (mneme):**
- manifold n=7 → BI 99.88
- metaculus n=2 → BI 99.95
- polymarket n=18 → BI 57.20

## Comparison across all three benches

Now that we have three runs, the trend is consistent:

| bench | n | freeze date | resolution window | crowd BI | mneme BI | paired Δ BI |
|---|---|---|---|---|---|---|
| 003 (single-shot, λ=0.2, no Platt) | 20 | 2024-07-21 | 2024-Q3 | 70.2 | 64.6 | −5.6 |
| 005 (iterative, paper-aligned) | 94 | 2024-07-21 | 2024-Q3-2026-Q1 | 57.5 | 84.1 | +26.6 |
| **006 (held-out)** | 58 | 2026-03-15 | 2026-Q1-Q2 | 58.3 | 84.8 | **+26.5** |
| **007 (held-out, second window)** | 27 | 2026-03-29 | 2026-04-08–29 | 11.0 | 71.4 | **+60.5** |

The +26-60 BI delta range across both held-out windows is the
durable finding. The bench-007 crowd-BI of 11 is dramatically lower
than bench-006's, suggesting the 2026-03-29 question_set's first 27
post-cutoff resolutions happened to be questions where the crowd was
particularly off-base. n=27 is small enough that one or two extreme
crowd misses dominate.

## Operational notes

- **0 failures.** The lenient EvidenceItem parser + MNEME-29 cleanup
  recovery (deployed in this build) eliminated the schema-violation
  failure mode that caused 5/63 in bench-006 and 6/100 in bench-005.
- **Wall-clock**: 22 min for 27 questions at concurrency=6, K=2, T=5.
  Effective throughput similar to bench-005.

## Caveats (still apply, restated)

1. **Web search is a contamination vector** even on post-cutoff
   questions. Resolutions in 2026-04-08 to 2026-04-29 generate news
   articles that the iterative loop's WebSearch can retrieve. The
   cleanest defense is BLFX-9 (4-layer date-leakage defense). Not yet
   built.
2. **Sample size for bench-007 is small** (n=27). The +60 BI point
   estimate is dramatic but the CI on the paired delta is wide; the
   per-source bins (n=2 for metaculus, n=7 for manifold) are
   borderline-meaningless on their own.
3. **Crowd BI 11 on bench-007 is suspicious.** Either the crowd was
   genuinely bad on this 27-question slice, or there's a selection
   effect (the post-cutoff resolutions in this window happen to be
   questions where the freeze price was unusually wrong). Worth
   checking against a third window.

## Decision

The algorithm + infrastructure work — that's settled. **The cleanest
remaining experiment is BLFX-9 (date-leakage defense): rerun the same
sample with a hardened "no information after freeze date" prompt and
see how much of the +26-60 BI delta survives.** That number is what
matters.

In parallel: the live marketplace pipeline (MNEME-30, also shipped
this session) is now collecting truly out-of-sample data — markets
that haven't resolved yet, frozen now, resolving over weeks. That
dataset will eventually produce the actual product number.

## Reproducibility

```bash
nohup python3 scripts/forecastbench_holdout_run.py \
  --question-set programs/_benchmarks/forecastbench/2026-03-29-llm.json \
  --resolution-sets \
    programs/_benchmarks/forecastbench/2026-03-29_resolution_set.json \
    programs/_benchmarks/forecastbench/2026-04-12_resolution_set.json \
  --resolve-from 2026-04-08 --resolve-to 2026-04-29 \
  --concurrency 6 --port 4456 --trials 2 --iterative-max-steps 5 \
  --output programs/_benchmarks/runs/<ts>-bench007-holdout-2/
```

Run dir: `programs/_benchmarks/runs/20260429-224759-bench007-holdout-2/`
