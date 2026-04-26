# BLFX-S04 Results: base LLM choice (Sonnet vs Gemini-3.1-Pro)

**Date:** 2026-04-26  
**Spike:** [BLFX-S04](../BLFX-S04.md)  
**Author:** Claude Opus 4.7 (analysis from prior knowledge; no live benchmarks)

## TL;DR

- **Use Sonnet for all development work.** It's 5x cheaper than Opus, ~5x cheaper than Gemini-3.1-Pro per million tokens, and within ~10% of frontier on the kind of reasoning + tool-use the BLFX loop requires.
- **Run the validation benchmark (BLFX-13) twice:** once on Sonnet (cheap, fast iteration), once on Opus (closer to paper's Gemini-3.1-Pro tier, defensible numbers).
- **Direct Gemini integration is out of scope for BLFX.** Adding a Gemini-backed claudecode would be a substrate change separate from this epic. Document the substitution; report numbers as "BLFX-on-Sonnet" or "BLFX-on-Opus" rather than "BLFX-on-Gemini-3.1-Pro."

## Context

The paper uses Gemini-3.1-Pro as BLF's base LLM (§3, paper note "we mostly use Gemini-3.1-Pro as our base LLM, but we evaluate other base models in section §4"). Our `claudecode` activation goes through the `claude` CLI, which uses Anthropic models. The question is whether substituting an Anthropic model invalidates our paper-comparison.

## Tier comparison (April 2026)

| Model | Provider | Tier | Reasoning bench (rough) | Tool-use reliability |
|-------|----------|------|------------------------|----------------------|
| Sonnet 4.6 | Anthropic | Mid | ~88% on common reasoning | High |
| Opus 4.7 | Anthropic | Frontier | ~92% on common reasoning | Very high |
| Gemini-3.1-Pro | Google | Frontier | ~91% on common reasoning | High |
| GPT-5 | OpenAI | Frontier | ~90% on common reasoning | High |

(These are rough estimates from public benchmark aggregations; exact numbers vary by benchmark and shift across model revisions. Treat as directional, not authoritative.)

## Cost (April 2026, public list pricing)

| Model | Input $/M | Output $/M | Notes |
|-------|----------|-----------|-------|
| Sonnet 4.6 | $3 | $15 | Anthropic API |
| Opus 4.7 | $15 | $75 | Anthropic API |
| Gemini-3.1-Pro | ~$15 | ~$60 | Google AI API |

For a BLFX run with K=5 trials × T_max=10 steps × 400 questions, expect ~20K input + 5K output tokens per trial (rough estimate; needs BLFX-S03 verification). That's:
- Sonnet: 400 × 5 × (20K × $3 + 5K × $15) per million → ~$270 for full benchmark
- Opus: 400 × 5 × (20K × $15 + 5K × $75) per million → ~$1,350 for full benchmark
- Gemini: similar to Opus tier, ~$1,200 estimated

## Recommendation

1. **All development on Sonnet.** Implementing BLFX-3 through BLFX-12 should target Sonnet by default. The cost difference is the key constraint; Sonnet supports reasonable iteration cycles.
2. **Spot-check ablations on Opus.** When BLFX-13 is being run, compute one ablation row on Opus to bound how much model tier accounts for any gap to the paper's BLF numbers.
3. **Final benchmark run on Opus.** When making the SOTA claim, use Opus. ~$1.4K per full run is acceptable for a single defensible benchmark.
4. **Do NOT block on Gemini integration.** Adding Gemini-3.1-Pro to claudecode would require a new model backend (claudecode currently shells out to the `claude` CLI which is Anthropic-only). That's a separate epic. We report numbers with our chosen model and document the substitution.

## Known limits of this spike

- **No live benchmarks.** This writeup is based on prior knowledge of model capabilities; we have not measured Sonnet-vs-Opus on actual BLFX runs. Once BLFX-4 (iterative loop) lands, we should measure 5 questions on each tier and update this writeup with empirical Brier deltas.
- **Tool-use reliability differences are anecdotal.** Both Sonnet and Opus handle native tool-use well in our experience, but there could be subtle differences in how they handle BLFX's iterative loop (especially the structured belief state output) that only show up at scale.
- **Pricing is volatile.** Anthropic and Google adjust pricing periodically; re-check at the time of BLFX-13.

## Open question for the user

Anthropic API or Claude subscription auth? The current substrate uses `claude login` (subscription). If we run a 400-question benchmark, the subscription rate limits will matter. May need to switch to direct API key + per-call billing for the benchmark run. Worth investigating before BLFX-13.

## Status

This spike's question is answered. Recommendation: Sonnet for dev, Opus for benchmark, document substitution. Spike closed; the open empirical follow-up (Sonnet-vs-Opus on real BLFX runs) lands as part of BLFX-13's methodology section.
