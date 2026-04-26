---
id: BLFX-16
title: "Wire execute_action to real tool backends (WebSearch first)"
status: Ready
type: implementation
blocked_by: [BLFX-4]
unlocks: [BLFX-13]
confidence: medium
---

## Problem

`agent_loop::execute_action` returns `Observation::Error("not yet implemented")`
for every action except `Submit`. After BLFX-4 wired `iterative_trial`
end-to-end against claudecode (Phase 1 — see
`plans/BLFX/results/BLFX-4-wire-phase1.md`), the loop scaffold works but
the model receives stubbed observations every step, so it mostly submits
early using training-only knowledge. The iterative loop's claimed -3.8 BI
benefit (Murphy 2026 ablation) cannot be realized until actions actually
fetch information.

## Context

The substrate already holds an `Arc<ClaudeCode<P>>`. `execute_action` needs
to dispatch each `Action` variant to a real backend:

| Action | Real backend |
|---|---|
| `Submit { p }` | already terminal — no fetch |
| `WebSearch { query, k }` | spawn / reuse a substrate-managed claudecode session with `WebSearch` tool, run a chat that returns a JSON list of `SearchHit` |
| `LookupUrl { url }` | claudecode session with `Read` / `WebFetch`, return page content |
| `SummarizeResults { result_ids }` | substrate-side: filter prior session's `SearchHit`s, call a tiny summarizer (one chat call) |
| `FetchTimeSeries { source, key }` | BLFX-15 ticket: real source-specific clients |
| `FetchWikipediaSection { article, section, as_of }` | BLFX-15 ticket: Wikipedia REST + revision lookup |

This ticket lands `WebSearch` (highest-value, simplest to wire) and
`LookupUrl`. The source-specific tools route through BLFX-15.

## Architecture

The cleanest place for this is the `StepDriver` trait — give it an
`execute_action` method that the loop calls instead of the standalone
`agent_loop::execute_action`. The mock driver returns canned observations;
`ClaudecodeStepDriver` makes real claudecode calls.

```rust
#[async_trait]
pub trait StepDriver: Send {
    async fn next_step(&mut self, ctx: StepContext<'_>) -> Result<String, String>;
    async fn execute_action(&mut self, action: Action) -> Observation;
}
```

This keeps `iterative_trial` driver-agnostic and removes the standalone
`execute_action` function (which can be left as a `Default` implementation
returning the stub for tests).

## Required behavior

For `WebSearch { query, k }`:
1. Open or reuse a substrate-managed search session (likely one per
   trial, distinct from the iterative session, allowed_tools=["WebSearch"]).
2. Send a prompt: "Search the web for: {query}. Return up to {k} results
   as a JSON array `[{ id, url, title, snippet, published_at }]`. Use
   the WebSearch tool exactly once."
3. Drain ChatEvents; capture the assistant's final text.
4. Parse the JSON into `Vec<SearchHit>`.
5. Return `Observation::SearchResults { results }`.
6. Accumulate the search session's token usage into the trial's running
   `TrialUsage`.

For `LookupUrl { url }`:
1. Same pattern with `Read` or `WebFetch`. Return `Observation::PageContent`.

Failure modes return `Observation::Error { message }` so the loop continues
gracefully (the model can pick a different action next step).

## Acceptance criteria

1. `StepDriver::execute_action` exists with default impl returning the
   stub observations (so the QueueStepDriver tests still pass unchanged).
2. `ClaudecodeStepDriver::execute_action` invokes real WebSearch via a
   spawned claudecode session and returns `SearchResults` with at least
   one populated `SearchHit`.
3. Live bench (`BLFX-4-WIRE-004`, T_max=5, K=1, iterative): the trial
   uses `> 3` turns and the artifact's `evidence_for` / `evidence_against`
   contain at least one item whose `source` is a real URL (not "training"
   or "user-provided").
4. Trial-level `usage_total.num_turns` > number of iterative steps — i.e.,
   the search sub-sessions' turns are accumulated correctly.
5. `cargo test --lib` passes.

## Out of scope

- `SummarizeResults` (depends on persistent SearchHit storage between
  steps — separate small ticket).
- `FetchTimeSeries`, `FetchWikipediaSection` — those are BLFX-15.
- Caching / dedup of repeated WebSearch queries.

## Completion

Tests pass + acceptance #3 verified live; status flips to `Complete`.
