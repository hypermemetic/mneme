---
id: MNEME-S03
title: "Spike: Parallel-fork scaling bound on substrate"
status: Pending
type: spike
blocked_by: []
unlocks: [MNEME-4]
confidence: n/a
---

## Question

How many concurrent `claudecode.chat` invocations can a single substrate process handle before degrading (latency spike, OOM, file-descriptor exhaustion, or arbor-tree write contention)?

The MNEME-4 `swarm.trial` design fans out N forks of a parent session and awaits all of them. If the substrate caps useful concurrency below ~10, swarm.trial must serialize internally or the BLF cost model breaks (one trial sweep takes 5×N×model-latency rather than 5×model-latency).

## Setup

1. Pre-create a `claudecode` session named `bench-parent` with a small system prompt.
2. Write a small driver (Rust binary or shell + jq) that:
   - Forks `bench-parent` into K children: `bench-K-1`, `bench-K-2`, ..., `bench-K-K`.
   - Issues `chat` to all K children concurrently with the same prompt: "Reply with the integer 42 and nothing else."
   - Times wall-clock from first issue to last completion.
3. Run for K = 1, 2, 4, 8, 16, 32. Record:
   - Wall-clock time per K
   - Per-call latency histogram (min/median/p99)
   - Substrate process RSS (`ps -o rss= -p <pid>`)
   - Substrate open FD count (`lsof -p <pid> | wc -l`)
   - Any errors in the substrate's stderr
4. Run each K twice; report the second run (warmup amortized).

## Pass condition

Wall-clock at K=8 is ≤ 1.5× wall-clock at K=1, AND no errors observed at any K up to 16. (8 is the target operating point for `swarm.trial`; 16 is the comfort margin.)

## Evidence to record

- The full table (K vs. wall-clock vs. p99 latency vs. RSS vs. FD count).
- The K at which wall-clock per call starts to grow superlinearly.
- The K at which the substrate emits the first error.
- The dominant cost: is the substrate the bottleneck, or is it Anthropic API rate limits?
- Whether the arbor tree write path serializes (look for mutex contention in `executor.rs` / arbor module).

## Fail → next

S-03b: if substrate is the bottleneck (not API rate limits), profile to identify the contention point. Likely candidates: arbor tree mutex, session map lock, JSON serialization in the loopback path.

## Fail → fallback

Cap `swarm.trial` at the K-value where wall-clock-per-call doubles; default N for `forecast` becomes that cap. If the cap is K < 3, multi-trial is not practical on a single substrate — split fan-out across multiple substrate processes (post-MVP, not in scope for phase 1).
