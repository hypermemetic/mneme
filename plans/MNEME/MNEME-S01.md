---
id: MNEME-S01
title: "Spike: Loopback MCP supports tool registration with typed JSON-schema input"
status: Pending
type: spike
blocked_by: []
unlocks: [MNEME-3]
confidence: n/a
---

## Question

Can the substrate's loopback MCP register a tool whose input shape is constrained by a JSON Schema such that Claude Code (running inside a `claudecode` session with loopback enabled) is required to call that tool with a payload matching the schema?

The MNEME-3 design depends on a "respond" tool registered per-skill, with the skill's output schema as the tool's input schema. The session is constrained via `disallowed_tools` so that Claude can't terminate without calling `respond` — and the tool-call payload IS the structured response.

This pattern works only if the loopback MCP exposes (a) tool registration with arbitrary JSON Schema and (b) Claude Code respects schema validation on tool inputs.

## Setup

1. Read `mneme-substrate/src/activations/claudecode/activation.rs` around the loopback handling code (look for `loopback_session_id` references and the MCP wiring).
2. Identify the existing loopback MCP server entry point — the code that responds to MCP `tools/list` and `tools/call`.
3. Build a minimal proof: register a tool `echo_typed` with input schema `{"type":"object","properties":{"value":{"type":"integer","minimum":0,"maximum":10}},"required":["value"]}`. Start a `claudecode` session with loopback enabled and a system prompt instructing Claude to call `echo_typed`.
4. Send `chat` with prompt: "Please call echo_typed with value 7."
5. Inspect the resulting `ChatEvent::ToolUse` — does Claude's payload conform to the schema?
6. Send a second prompt: "Please call echo_typed with value 'seven' (the string)." Does the call get rejected somewhere (Claude refuses, MCP rejects, substrate rejects), or does an invalid payload reach the tool handler?

## Pass condition

Both of the following observed:
- (a) Valid call: `ToolUse { tool_name: "echo_typed", input: {"value": 7} }` reaches the substrate handler.
- (b) Invalid call: the substrate handler does NOT receive a payload with `value: "seven"` — either Claude self-corrects after a tool result error, or MCP layer enforces the schema.

## Evidence to record (regardless of pass/fail)

- The exact mechanism Claude Code uses to learn tool schemas (does it pass them to the model? request `tools/list` once? per turn?).
- Whether `disallowed_tools` exists / works to forbid `Stop` / `Task` etc., forcing Claude to keep going until `respond` is called.
- Latency of the round-trip (relevant to `swarm.trial` cost).
- Any error modes — e.g., Claude calling a different tool, looping, failing to call any tool.

## Fail → next

S-02-alt: drop the schema-enforcement requirement; rely on prompt-engineered JSON inside `Content` events plus a parsing retry loop in the skill activation. Record the parse-failure rate over 20 trials of `forecast` to quantify the cost.

## Fail → fallback

If even prompt-engineered JSON fails reliably: structured outputs become best-effort with a typed-but-optional shape. Skill activations return `Result<T, ParseError>` and mneme retries up to a bound. Forecast confidence drops one bucket on parse retry.
