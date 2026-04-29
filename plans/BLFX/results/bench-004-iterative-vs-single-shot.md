---
title: "bench-004 — paired iterative-vs-single-shot at n=20 (paper-aligned config)"
date: 2026-04-29
status: complete
---

## Headline

| Run | mean Brier | Brier Index | 95% CI on mean Brier | Δ vs crowd |
|---|---|---|---|---|
| bench-003 (λ=0.2, no Platt, single-shot) | 0.0884 | 64.65 | [0.0041, 0.1967] | −5.58 |
| **bench-004a** (λ=0.0, Platt, single-shot) | **0.0786** | **68.55** | [0.0206, 0.1633] | −1.68 |
| **bench-004b** (λ=0.0, Platt, iterative T_max=5) | **0.0410** | **83.59** | [0.0091, 0.0867] | **+13.36** |
| Crowd baseline (paired, same n=20) | 0.0744 | 70.23 | [0.0298, 0.1307] | — |

**Observations**:

1. Just **paper-aligning the algorithm defaults** (λ=0.2 → 0.0 + Platt
   calibration applied) lifted BI from 64.65 → 68.55 (**+3.9 BI**) with
   zero new algorithmic work. MNEME-26/27/28 alone closed 70% of the
   crowd gap.
2. Adding the **iterative loop** lifted BI further from 68.55 → 83.59
   (**+15.0 BI**), putting mneme **above the crowd by +13.36 BI** on
   this paired sample. Paper claimed +3.8 BI from iterative; we got
   ~+15 in our setup.
3. **Caveat dominates the headline.** The +15 BI delta is largely a
   single question (xv86) where iterative caught what single-shot
   missed — Brier went 0.633 → 0.005, a 0.628 swing. Strip that
   question out and the remaining 19 deltas are mostly small or
   slightly in single-shot's favor.

## Paired analysis (the honest read)

Per-question Brier delta (iterative − single-shot), paired bootstrap on
mean delta with 10000 resamples:

- Mean paired delta on Brier: **−0.0376** (iterative wins by ~0.04)
- Mean paired delta on BI:    **+15.04**
- **95% CI on paired mean Brier delta: [−0.1048, +0.0012]**
- The CI **barely includes 0** on the upper edge.

Decision per BLFX-17's pre-registered rules:
- "If delta BI ≥ +2 with CI excluding 0: flip default"
- "If delta BI ∈ [−2, +2]: re-run at higher n to disambiguate"
- "If delta BI ≤ −2 with CI excluding 0: iterative is hurting"

Strictly interpreted, the CI doesn't decisively exclude 0 (lower bound
+0.0012 ≈ −0.5 BI in the worst case). So the rule says **do not flip
the default yet**. n=100 (BLFX-18) is the disambiguator.

## Per-question table

| qid | source | actual | ss p | it p | ss Brier | it Brier | Δ |
|---|---|---|---|---|---|---|---|
| 0x0590ee... | polymarket | 0.0 | 0.463 | 0.487 | 0.2148 | 0.2372 | +0.0224 |
| 0x17620f... | polymarket | 0.0 | 0.062 | 0.068 | 0.0039 | 0.0046 | +0.0007 |
| 0x404310... | polymarket | 0.0 | 0.051 | 0.070 | 0.0026 | 0.0049 | +0.0023 |
| 0x438883... | polymarket | 0.0 | 0.066 | 0.062 | 0.0044 | 0.0039 | −0.0005 |
| 0x598620... | polymarket | 0.0 | 0.042 | 0.042 | 0.0017 | 0.0017 | 0.0000 |
| 0x60752c... | polymarket | 1.0 | 0.766 | 0.912 | 0.0546 | 0.0078 | **−0.0468** |
| 0xaadfaf... | polymarket | 0.0 | 0.021 | 0.023 | 0.0004 | 0.0005 | +0.0001 |
| 0xb122e0... | polymarket | 0.0 | 0.092 | 0.062 | 0.0085 | 0.0039 | −0.0046 |
| 0xf58754... | polymarket | 0.0 | 0.062 | 0.062 | 0.0039 | 0.0039 | 0.0000 |
| 1374 | metaculus | 1.0 | 0.622 | 0.784 | 0.1427 | 0.0468 | **−0.0959** |
| 15796 | metaculus | 1.0 | 0.376 | 0.386 | 0.3893 | 0.3772 | −0.0121 |
| 24034 | metaculus | 0.0 | 0.092 | 0.092 | 0.0085 | 0.0085 | 0.0000 |
| 3wRmZN... | manifold | 1.0 | 0.881 | 0.893 | 0.0142 | 0.0114 | −0.0028 |
| 7664 | metaculus | 1.0 | 0.881 | 0.805 | 0.0142 | 0.0379 | +0.0236 |
| AXdiGo... | manifold | 0.0 | 0.123 | 0.136 | 0.0152 | 0.0184 | +0.0032 |
| KCFbp1... | manifold | 1.0 | 0.841 | 0.871 | 0.0252 | 0.0166 | −0.0086 |
| KtGRsT... | manifold | 0.0 | 0.126 | 0.078 | 0.0158 | 0.0061 | −0.0097 |
| PNVVTl... | manifold | 1.0 | 0.893 | 0.872 | 0.0114 | 0.0165 | +0.0051 |
| VB1RhU... | manifold | 1.0 | 0.912 | 0.912 | 0.0078 | 0.0078 | 0.0000 |
| **xv86CD...** | manifold | 0.0 | **0.796** | **0.070** | **0.6332** | **0.0049** | **−0.6284** |

The xv86 row is the swing case. Without it, the next biggest delta in
iterative's favor is question 1374 at −0.096. With xv86 included, the
mean delta is dominated by one observation.

## What changed on xv86 between modes

Question xv86 (manifold): "Will an artificial intelligence system play
a major artistic role in the 2024 Olympic opening ceremonies?"
freeze_value=0.133, resolved NO.

- Single-shot (004a): predicted 0.796 (Brier 0.633) — **the model
  read the headline-style framing differently and got the answer
  wrong**, possibly mis-resolving the question.
- Iterative (004b): predicted 0.070 (Brier 0.005) — the iterative
  loop's WebSearch sessions evidently surfaced the resolution criteria's
  fine print + the actual ceremony's lack of explicit AI integration,
  which the single-shot read missed.

This is the kind of question where the iterative loop's structural
advantage shows: a chance to web-search the actual resolution facts
rather than reasoning from a one-line description. Worth verifying
on more samples in BLFX-18.

## Cost / wall-clock

- bench-004a (single-shot K=2 n=20 concurrency=4): **438 s = 7.3 min**
- bench-004b (iterative T_max=5 K=2 n=20 concurrency=4): **571 s = 9.5 min**

Iterative cost was ~30% over single-shot in wall-clock — much less
than expected (would have been 5× without parallelism). MNEME-24/25
(parallel trials + question-level concurrency) make iterative
operationally tractable.

## Caveats (the part to read carefully)

1. **Test-set contamination is severe.** All 20 questions resolved
   between 2024-07-25 and 2024-09-04. Sonnet's training cutoff is well
   after that — the model knew which Russian athletes competed at
   Paris 2024, what happened with Bitcoin, etc. The iterative loop's
   WebSearch makes this WORSE, not better, because search retrieves
   articles that explicitly discuss the resolution. **The +13 BI vs
   crowd is not a fair benchmark of the algorithm**; it's a benchmark
   of "model + web with answer keys vs market that froze before
   answers existed."
2. **n=20 has wide CI.** 95% CI on iterative's mean Brier alone is
   [0.009, 0.087], which translates to BI [85, 65]. The 83.59 point
   estimate could easily move ±10 BI on a different sample.
3. **One question dominates.** Strip xv86 and the paired mean delta
   shrinks dramatically.
4. **Selection bias.** First 20 questions by resolution-date skews
   toward shorter-term markets, which are also the easiest to look
   up after-the-fact.

## Decision and next steps

- **Do NOT flip default `iterative_max_steps` to Some(5) yet.**
  The CI on the paired delta barely includes 0; we should disambiguate
  at n=100 first.
- Run BLFX-18 (bench-005) at n=100 with the iterative path active.
  At n=100 the same effect size (mean Brier delta ≈ −0.04) would
  produce a CI roughly [−0.07, −0.01], which excludes 0 cleanly.
- Once BLFX-18 confirms, flip the default.
- For honest external claims, we need **post-cutoff held-out
  questions** (2026-Q3+ resolutions). The current numbers say "the
  algorithm + machine works"; they don't say "we beat the crowd on
  truly unseen questions."

## Build / commit references

- Substrate at MNEME-28 commit (Platt wired) — calibration_applied log
  fired live during both runs (a=0.5947, b=−0.3998).
- DEFAULT_LAMBDA = 0.0 per MNEME-27.
- Calibration store seeded with 20 obs per MNEME-26.
- bench-004a results: `programs/_benchmarks/runs/20260429-125553-bench004a-singleshot-lambda0/`
- bench-004b results: `programs/_benchmarks/runs/20260429-130411-bench004b-iterative/`
