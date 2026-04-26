# Benchmark 001: K=5 baseline timing

**Date:** 2026-04-26  
**Question:** "Will the S&P 500 close above 6,000 by end of December 2026?"  
**Substrate:** mneme-substrate on port 4456, model=sonnet  
**Program:** `c868766a-e107-45fe-ab6c-0c4058cc721e`  
**Settings:** trials=5, parent_session=forecast-fresh-session, allowed_tools=["WebSearch"]

## Wall-clock

| Started | Finished | Elapsed |
|---------|----------|---------|
| 10:43:17 | 10:48:00 | **4m 43s** |

## Comparison to K=2 baseline

| Trial count | Wall-clock | Wall-clock per trial |
|-------------|-----------|----------------------|
| 2 | 65s | 32.5s |
| 5 | 4m 43s (283s) | **56.6s** |

**Per-trial time grew 1.7x going from K=2 to K=5.** If trials were truly independent and parallel, per-trial time should be roughly constant. The growth suggests one or more of:

1. **Substrate contention.** Forking 5 sessions from a shared parent, all writing to arbor concurrently, may serialize on the arbor write lock or sqlite pool.
2. **Anthropic API rate limiting.** 5 simultaneous chats from one account may hit per-minute concurrent-request limits, queuing some.
3. **Resource competition.** Local CPU/memory pressure under 5 concurrent claude CLI processes.

Worth probing in BLFX-S03 (cost spike): which of (1)/(2)/(3) is dominant.

## Cost (rough estimate)

Did not capture token counts directly (need to instrument). Rough estimate:
- 5 trials × ~5K input + 2K output tokens (Sonnet)
- Sonnet: $3/M in + $15/M out
- ≈ 5 × ($0.015 + $0.030) ≈ **$0.23 per K=5 forecast**

For full BLFX-13 benchmark (400 questions × K=5):
- ≈ 400 × $0.23 = **$92** on Sonnet without iterative loop
- With BLFX-4 iterative loop (T_max=10): ≈ 5–10x more steps per trial → **$500-1000 plausible** on Sonnet
- On Opus (5x cost): **$2.5K-5K plausible**

These are rough; **BLFX-S03 spike should run a small N-question sample with token-counting instrumentation and produce a defensible estimate** before committing to a full benchmark run.

## Findings

1. **Per-trial timing grows non-linearly with K.** Investigate whether the substrate, the API, or the local resources is the bottleneck.
2. **Token instrumentation is missing.** The substrate doesn't currently record total tokens per program. Adding this is required for BLFX-S03.
3. **Wall-clock at K=5 is ~5 minutes per question.** Even at 100% parallelism (which we don't have), 400 questions = 33 hours. Need concurrency-across-questions in BLFX-12 (backtest runner) to make benchmarking feasible.

## Action items

- [ ] BLFX-S03 spike: instrument token counts; run 5-question sample with timing + token capture; produce $/question estimate
- [ ] BLFX-12 ticket should specify concurrency-across-questions (not just within a question)
- [ ] Open a follow-up ticket for "investigate K-scaling bottleneck"
