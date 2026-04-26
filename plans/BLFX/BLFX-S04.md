---
id: BLFX-S04
title: "Spike: base LLM choice (Sonnet vs Gemini-3.1-Pro)"
status: Complete
type: spike
blocked_by: []
unlocks: [BLFX-4]
confidence: n/a
---

## Question

The paper uses Gemini-3.1-Pro as BLF's base LLM. We have Claude Code (Sonnet by default) wired through claudecode. Is Sonnet (or Opus) close enough to Gemini-3.1-Pro on forecasting tasks that our benchmark numbers will be defensibly comparable to the paper's?

## Setup

1. Look up published evaluations of Gemini-3.1-Pro vs Claude Sonnet/Opus on reasoning + tool-use benchmarks (MMLU, GPQA, MATH, SWE-Bench, etc.).
2. Look at the paper's section §4 — they evaluate other base LLMs. Do they report Sonnet or Opus numbers? Section 4 referenced from §3 of the paper as having "results on AIBQ2."
3. Cost difference: Sonnet vs Opus vs Gemini-3.1-Pro per million tokens.
4. Tool-use reliability: any known differences in how each model handles iterative tool-use loops.

## Pass condition

Either:
- (A) Sonnet is within ~10% on relevant benchmarks → use Sonnet, document the substitution.
- (B) Sonnet is meaningfully worse → recommend running on Opus (more expensive but closer to Gemini-3.1-Pro tier) OR add Gemini API integration to claudecode (out of scope for this epic).

## Evidence to record

- Public benchmark numbers for each model on reasoning + tool-use.
- Whether the paper's §4 evaluates Sonnet/Opus directly.
- Cost ratio Sonnet:Opus:Gemini-3.1-Pro per token.
- Whether claudecode supports model-per-session selection (it does — model is a `Create` parameter).

## Fail → next

If Sonnet underperforms badly: explore whether Opus is cost-feasible per BLFX-S03's findings. If yes, switch to Opus.

## Fail → fallback

If neither Sonnet nor Opus reaches Gemini-3.1-Pro: report numbers as "BLFX-on-Sonnet" rather than direct paper-comparison; document the model gap as a known confound; add a future ticket for Gemini API integration if pursuing direct paper-comparable numbers.
