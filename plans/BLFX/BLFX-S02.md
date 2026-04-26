---
id: BLFX-S02
title: "Spike: action format for the iterative loop"
status: Ready
type: spike
blocked_by: []
unlocks: [BLFX-3, BLFX-4]
confidence: n/a
---

## Question

For the BLFX iterative agent loop, should each step use Claude's native tool-use protocol (the model picks a tool from a registered set, the substrate executes, returns a tool_result), OR a custom JSON action protocol (the model emits `{action: web_search, args: {query: ...}, belief: {...}}` as structured text in its response, the substrate parses)?

## Setup

1. Read claudecode's `chat()` event stream — what tool-use events does it emit when a tool is registered? Does it require the tool to be MCP-registered, or can it be inline?
2. Try a tiny experiment: register a fake `web_search` tool via the loopback MCP, send a chat asking the LLM to call it, observe the round-trip latency and reliability.
3. Try the alternative: prompt the LLM to emit `{action: ..., args: ..., belief: ...}` JSON, parse it, execute, send the result back as a user message.
4. For each, estimate: latency per step, parse/dispatch reliability, ease of belief-state update integration.

## Pass condition

We have a clear winner. Specifically, EITHER:
- (A) Native tool-use is reliable and supports inline belief-state updates → use it.
- (B) Custom JSON is more reliable for our combined (action + belief) shape → use it.

## Evidence to record

- Round-trip latency for one tool call via each protocol.
- Parse failure rate on N=20 trials of each.
- Whether the model can produce the COMBINED `(action, belief)` output cleanly in one response (the paper's algorithm needs both per step).
- Whether tool registration is per-call or per-session (affects whether we register once per trial or per step).

## Fail → next

If both protocols are unreliable: build a hybrid — native tool-use for the action, follow-up message for the belief state. Two LLM calls per step instead of one. Document the cost.

## Fail → fallback

If reliability is bad enough that the iterative loop is impractical: stay single-shot per trial but do K=5 trials with diversification. Accept the ablation hit (~3.8 BI per the paper) as a known limit.
