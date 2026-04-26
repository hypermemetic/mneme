---
id: BLFX-S03
title: "Spike: cost estimate for one full BLFX benchmark run"
status: Complete
type: spike
blocked_by: []
unlocks: [BLFX-13]
confidence: n/a
---

## Question

What is the realistic dollar cost (Anthropic API + tool API) of running one full BLFX benchmark on 400 ForecastBench questions, K=5 trials per question, T_max=10 steps per trial?

## Setup

1. Sample 5 representative questions across question types (market binary, market multi-resolution, dataset numerical, dataset judgemental).
2. Run each through the full BLFX loop (assuming BLFX-4 is implemented OR a single-shot proxy with tools).
3. Sum input + output tokens per trial; average across trials per question; multiply by K=5.
4. Multiply by 400 questions.
5. Apply Anthropic pricing for the chosen model (Sonnet vs Opus).
6. Add Brave Search (or whatever WebSearch backend) cost: paper says default tool budget is ~10 actions, mostly searches.

## Pass condition

Total estimated cost ≤ $200 per benchmark run. (Reasoning: this is the kind of one-shot research expense that a small team should be able to absorb for validation purposes; anything more starts requiring grant-level justification.)

## Evidence to record

- Average input tokens per trial step.
- Average output tokens per trial step.
- Average trial steps to convergence (do trials usually `submit` before T_max or hit the cap?).
- Per-question total cost (low / median / high).
- Estimated total benchmark cost at chosen model.

## Fail → next

S-03b: estimate cost on Sonnet-3.5 (cheaper) or with K=3 trials (less aggregation power but lower cost). Find a (K, model) point that's both affordable and not too far from the paper's K=5.

## Fail → fallback

If full benchmark is unaffordable: run on 50 questions (one-eighth of the paper's). Report that we evaluated on a smaller subset. Note that bootstrap CIs will be wider but the directional answer (does our BLFX match the paper?) should still be defensible.
