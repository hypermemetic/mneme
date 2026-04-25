---
id: MNEME-S02
title: "Spike: Fork independence under shared system prompt"
status: Pending
type: spike
blocked_by: []
unlocks: [MNEME-4]
confidence: n/a
---

## Question

When `claudecode.fork` produces N child sessions from the same parent (same model, same system prompt, same conversation tree position), do the children produce **independent reasoning paths** when given the same prompt, or are they **strongly correlated**?

BLF multi-trial assumes independence: the variance across trials is what aggregation reduces. If forks are correlated (e.g., temperature is too low, or sampling seed is shared), multi-trial degenerates into N×cost for the same answer.

## Setup

1. Create a `claudecode` session with model `sonnet` (cheaper for this experiment), no system prompt, working dir `/tmp/mneme-spike`.
2. Send one priming prompt to establish a non-trivial conversation state: "I'm going to ask you a forecasting question. Reason step by step before giving a probability."
3. Fork the session 5 times.
4. To each fork, send the same prompt: "Question: Will Bitcoin close above $100,000 USD on 2026-12-31? Reason step by step, then give your final probability as a decimal between 0 and 1 on its own line prefixed with `P=`."
5. Capture the final `P=` value from each fork's `Content` stream.
6. Repeat with N=5 forks of a fresh session three times (total 15 forks).

## Pass condition

The 15 probabilities have **standard deviation ≥ 0.05** (i.e., trials disagree by at least 5 percentage points on average). Lower std-dev means forks are too correlated for BLF to do useful work.

## Evidence to record

- The 15 probability values, organized by trial-set.
- A qualitative read of the reasoning chains: do trials cite different evidence, or is the prose nearly identical with a different number tacked on?
- Whether the model id matters (rerun with `opus` or `haiku` if `sonnet` shows zero variance).
- Whether the priming-then-fork pattern matters vs. forking before any prompts (the latter forces the model to set up its own context per fork — possibly more diverse).

## Fail → next

S-02b: try injecting a per-trial diversification clause into the system prompt at fork time ("You are reviewer #N of 5; emphasize the [bull/bear/base] case in your reasoning"). Re-measure std-dev.

## Fail → fallback

If even diversification clauses don't push std-dev above 0.05: BLF multi-trial degenerates to single-trial-with-confidence-tagging in this substrate. Skills using `swarm.trial` default to N=1 and emit `confidence: single-pass` in the result. Document this as a known limit; aggregation skill methods become no-ops that pass through the single trial.
