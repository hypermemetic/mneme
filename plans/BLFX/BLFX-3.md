---
id: BLFX-3
title: "Action enum + parser for the iterative loop"
status: Complete
type: implementation
blocked_by: [BLFX-S02]
unlocks: [BLFX-4]
confidence: medium
---

## Problem

The paper's BLF agent loop (Algorithm 1) selects one of several actions at each step: `web_search`, `summarize_results`, `lookup_url`, source-specific data tools (`fetch_ts_yfinance`, `fetch_wikipedia_section`, ...), and `submit`. We need a typed Action enum, an executor for each variant, and a parser that turns the LLM's output into a typed Action.

## Context

BLFX-S02 picks the action format (Claude native tool-use vs custom JSON). This ticket implements the chosen format. Either way, we need:

- A typed `Action` enum representing every supported tool.
- A `parse_action(llm_output) → Result<Action, ParseError>` function.
- An `execute_action(action, env_context) → Observation` function.
- The loop in BLFX-4 calls these in sequence.

Source-specific tools (`fetch_ts_yfinance`, etc.) are the paper's specialty; we start with the universal subset and add source-specific tools per-source as needed.

## Evidence

The paper's §2 lists their tool set explicitly. Implementing the universal subset (`web_search`, `summarize_results`, `lookup_url`, `submit`) covers most question types. Source-specific tools matter for time-series questions (yfinance, FRED, DBnomics) and Wikipedia-grounded questions; those can be added later as separate tools without changing the loop architecture.

The choice of Claude tool-use vs custom JSON (BLFX-S02) determines whether `Action` is a tagged enum we serialize/deserialize OR a Claude tool definition. Either way the variants are the same.

## Required behavior

```rust
pub enum Action {
    WebSearch { query: String, k: u8 },                  // top-k search results
    SummarizeResults { result_ids: Vec<String> },        // filter + summarize prior search results
    LookupUrl { url: String },                           // fetch + read a specific URL
    FetchTimeSeries { source: String, key: String },     // yfinance, FRED, etc.
    FetchWikipediaSection { article: String, section: String },
    Submit { probability: f64 },
}

pub enum Observation {
    SearchResults { results: Vec<SearchHit> },
    Summary { text: String },
    PageContent { url: String, content: String, fetched_at: DateTime<Utc> },
    TimeSeries { source: String, key: String, points: Vec<(DateTime<Utc>, f64)> },
    WikipediaSection { article: String, section: String, content: String },
    Submitted { probability: f64 },
    Error { message: String },
}

pub fn parse_action(llm_output: &str) -> Result<(Action, ForecastState), ParseError>;
pub async fn execute_action(action: Action, env: &EnvContext) -> Observation;
```

Behavior:

| Scenario | Outcome |
|----------|---------|
| LLM output is valid action + belief | Returns `(Action, ForecastState)` |
| LLM output is malformed (no JSON, wrong shape) | Returns `ParseError`; loop retries with the parse error appended to history |
| Action is `Submit` | Loop terminates, returns the probability |
| Action requires a tool that's not in the registered subset | Returns `ParseError` with available actions listed |
| `WebSearch` query when offline | Observation::Error, loop continues |
| `LookupUrl` fetches a 404 | Observation::Error, loop continues |

## Risks

| Risk | Mitigation |
|------|-----------|
| LLM emits actions outside the registered subset | Parser validates against the registry; clear error sent back as next-turn user message |
| Source-specific tools (yfinance, etc.) require external API keys we don't have | Start without them; loop just doesn't pick them; add per-source as needed |
| `WebSearch` results need date-clamping for backtest validity | Defer to BLFX-9 (4-layer date defense); this ticket's `WebSearch` takes no cutoff date — the env wrapper handles it |

## What must NOT change

- `claudecode`'s public API.
- `MnemeContext`'s shape.

## Acceptance criteria

1. `Action` and `Observation` enums in `forecast/agent_loop.rs` (new module), `Serialize + Deserialize + JsonSchema`.
2. `parse_action` handles malformed LLM output gracefully (returns ParseError, doesn't panic).
3. `execute_action` for `WebSearch`, `LookupUrl`, `Submit`, `SummarizeResults` (the universal subset) implemented.
4. Source-specific tool variants are defined as enum members but their `execute_action` arms return `Observation::Error { message: "tool not implemented in this build" }`.
5. Unit tests: parse 10 example LLM outputs (5 valid, 5 malformed), execute each action against mocks.
6. `cargo test --lib` passes green.

## Completion

Tests pass; status flips to `Complete` in the same commit.
