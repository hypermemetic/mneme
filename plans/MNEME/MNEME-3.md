---
id: MNEME-3
title: "`respond` tool + structured-output protocol"
status: Pending
type: implementation
blocked_by: [MNEME-S01]
unlocks: [MNEME-4, MNEME-6]
confidence: medium
---

## Problem

A skill activation needs to return a typed value (e.g., `Forecast { probability: f64, summary: String, decomposition: Vec<SubQuestion> }`) but the underlying `claudecode.chat` produces only free-form `Content` text events. Without an enforcement mechanism, structured outputs are best-effort and parse failures become a surface area for callers.

## Context

- `claudecode` activation (`mneme-substrate/src/activations/claudecode/activation.rs`) supports `allowed_tools` / `disallowed_tools` parameters that pass through to the Claude CLI's `--allowedTools` / `--disallowedTools` flags.
- The substrate's loopback MCP exposes substrate-side tools to the running Claude session (mechanism investigated in MNEME-S01).
- Tool inputs in the Anthropic API are constrained by JSON Schema; Claude is trained to respect schemas and self-correct on validation errors.

## Evidence

MNEME-S01 establishes whether loopback MCP supports schema-constrained tools. **This ticket is blocked until S01 returns evidence.** Two paths depending on S01:

- **S01 passes (preferred path):** register a per-skill `respond` tool whose input schema is the skill's output schema. Constrain the session via `disallowed_tools = ["Stop"]` (or whatever forces continuation) so Claude must call `respond` before terminating. The tool-call payload IS the structured response. Validation happens at the tool boundary; invalid payloads bounce back to Claude with the schema error and Claude self-corrects.
- **S01 fails (fallback path):** abandon tool-based structured output. Use prompt-engineered JSON: instruct Claude in the system prompt to emit a single fenced `json` block as the last thing, parse with a regex, retry up to 3 times on parse failure.

The choice is binary based on S01 evidence — both are implementable, but the tool path is meaningfully more reliable (tool-use round-trip is a trained capability; "emit a single JSON block" is not). Confidence is `medium` because both paths might require non-trivial work in the substrate.

## Required behavior

A new module `mneme/src/respond.rs` (or wherever the harness lives) exposes:

```
fn register_respond_tool(
    session_name: &str,
    schema: serde_json::Value,    // JSON Schema for the response
) -> RespondToolHandle;

async fn await_response<T: DeserializeOwned>(
    handle: RespondToolHandle,
) -> Result<T, RespondError>;
```

Behavior:

| Scenario | Expected outcome |
|----------|-----------------|
| Claude calls `respond` with valid payload | `await_response` returns `Ok(T)` parsed from the payload |
| Claude calls `respond` with payload that fails schema validation | The substrate rejects via the tool result; Claude retries; if Claude fails 3 times, `await_response` returns `Err(RespondError::Validation { last_payload, last_error })` |
| Claude attempts to terminate without calling `respond` | The session's `disallowed_tools` prevents this; Claude must continue |
| Claude calls `respond` then continues talking | The first valid payload wins; subsequent calls are ignored |

The handle is auto-cleaned up when dropped (deregister the tool from the loopback MCP).

## Risks

| Risk | Mitigation |
|------|-----------|
| `disallowed_tools = ["Stop"]` doesn't actually exist as a Claude tool name | S01 evidence resolves this; if no such tool exists, replace with a system-prompt clause + retry-on-empty-response |
| Schema is too restrictive and Claude can't satisfy it (e.g., probability `[0, 1]` but Claude returns 1.0 as `1`) | Use generous types in the schema (`number` not `[exclusiveMinimum: 0, exclusiveMaximum: 1]`); validate semantic constraints in the skill, not in the tool schema |
| Loopback MCP tool registration has global state and concurrent skill calls collide | Tool name is suffixed with the program_id (`respond_<program_id>`); each program has its own |

## What must NOT change

- The `claudecode` activation API. This ticket consumes it as-is; if the API needs changes, they belong in a separate ticket against mneme-substrate (or upstream plexus-substrate if the change should backport).
- The `ChatEvent` enum. We piggyback on existing `ToolUse`/`ToolResult` events; no new event types.

## Acceptance criteria

1. `mneme/src/respond.rs` (or equivalent) implements `register_respond_tool` and `await_response`.
2. Integration test `tests/respond_basic.rs`: registers a tool with schema `{type: object, properties: {value: {type: integer}}, required: [value]}`, drives a `claudecode` session with prompt "call respond with value 42", asserts the returned value is `42`.
3. Integration test `tests/respond_invalid.rs`: same setup, but instructs Claude to call `respond` with `value: "not a number"`. Asserts the call retries and either succeeds (Claude self-corrects) or returns `Err(Validation { ... })` after 3 attempts.
4. Integration test `tests/respond_no_call.rs`: instructs Claude to "say hello and stop." Asserts `disallowed_tools` (or the fallback) prevents termination, Claude eventually calls `respond` or the call returns `Err(Timeout)` after a bounded wait (60s).
5. `cargo build` and `cargo test` pass green for the `mneme` crate.

## Completion

Implementor runs `cargo test --test respond_basic respond_invalid respond_no_call`, all three pass, status flips to `Complete` in the same commit as the code.

## Implementation status (2026-04-26 autonomous session)

`RespondTool` descriptor type and the per-program tool name scheme are in `mneme-substrate/src/mneme/respond/mod.rs`. JSON Schema validator (type/required/range) in `mneme-substrate/src/mneme/respond/schema.rs`. The `ToolRegistry` (per-program registry with RAII handle) is in `mneme-substrate/src/mneme/runtime/tool_registry.rs`. **What's missing:** integration with the loopback MCP server's `tools/list` and `tools/call` handlers — gated on MNEME-S01 spike confirming the loopback supports schema-constrained tool registration.
