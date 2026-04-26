---
id: MNEME-17
title: "Wire `mneme run` CLI subcommand to invoke skills end-to-end"
status: Pending
type: implementation
blocked_by: []
unlocks: []
confidence: medium
---

## Problem

The `mneme` CLI binary (`src/bin/mneme.rs`) has working `inspect`, `programs list`, and `programs trace` subcommands that read filesystem state. The fourth subcommand, `mneme run <method> [args...]`, is currently a stub that prints a "not yet implemented" message. With `forecast.update` now functional through the substrate, `run` becomes the natural one-command entry point for skill invocation.

## Context

- The substrate exposes skills via Plexus RPC over WebSocket on port 4444 (4456 in dev).
- Synapse already provides a generic schema-driven CLI for any Plexus backend. `mneme run` doesn't replace synapse; it bundles substrate startup + skill invocation into one process.
- The substrate construction in `builder.rs::build_plexus_rpc` is async and returns `Arc<DynamicHub>`. The same construction can run inside `mneme run` to invoke methods directly via Rust calls (bypassing the WebSocket round-trip).
- `mneme-substrate/Cargo.toml` already lists `mneme-substrate` as the library crate, so `mneme.rs` can `use mneme_substrate::build_plexus_rpc`.

## Evidence

Two design choices to make:

**(A) Bundled substrate (in-process, no WebSocket).** `mneme run forecast.update --program-id Q-001 ...` constructs `Arc<DynamicHub>` in-process, dispatches the call directly, prints the result, exits. Pro: one binary, no port management, no daemon. Con: each invocation pays the substrate startup cost (claudecode storage init, etc. — currently ~1-2s).

**(B) Connect to a running substrate.** `mneme run` is a thin synapse-equivalent that connects to `--port 4444` and forwards the call. Pro: cheap; can run many invocations against one warm substrate. Con: requires `mneme-substrate` to already be running.

Recommend **(A)** as the default for one-shot ergonomics, with `--connect <host:port>` to opt into (B) when a long-running substrate exists. The startup cost is real but small (under 2s in current testing) and only a one-shot user pays it.

The argument-parsing problem is real: `forecast.update` takes structured args (`--allowed-tools '["WebSearch"]'` failed with synapse). Approach: accept JSON via `--params '{...}'` for full structured input, plus shorthand flags (`--program-id <id>`) that the CLI translates into the params object before dispatch.

## Required behavior

`mneme run <skill.method> [--params <json> | --<key> <value> ...]`

Examples:
```
mneme run forecast.update --program-id Q-001 --new-evidence "BTC at $95k" --trials 2
mneme run forecast.update --params '{"program_id":"Q-001","new_evidence":"...","trials":2,"allowed_tools":["WebSearch"]}'
mneme run programs.list
```

Behavior:

| Scenario | Expected outcome |
|----------|-----------------|
| Method exists; args valid | Constructs substrate, dispatches, streams events to stderr (so stdout is parseable result), prints final result as JSON to stdout, exit 0 |
| Method doesn't exist | Prints available methods (from `DynamicHub::list_activations_info`); exit 1 |
| Args invalid (validation error from skill) | Prints the Error event from the stream; exit 1 |
| `--connect host:port` provided | Skips substrate construction; opens WebSocket to the given host; dispatches; prints |
| Method emits multiple events | Streams them as they arrive (one JSON object per line on stderr); final event becomes the result on stdout |

## Risks

| Risk | Mitigation |
|------|-----------|
| Substrate startup is slow enough to harm dev loop | Profile; if startup is over 5s, support `--connect` mode against a long-running substrate as the default for repeated calls |
| Argument parsing for nested objects is awkward | `--params <json>` is the escape hatch; document it for any complex shape |
| Streaming output to stderr while the result lands on stdout is non-standard for CLIs | Document; mirror synapse's `--json` flag if needed |

## What must NOT change

- The existing `inspect`, `programs list`, `programs trace` subcommands.
- The substrate's external API.

## Acceptance criteria

1. `mneme run forecast.update --program-id Q-001 --new-evidence "test" --trials 2` (with the substrate-binary unset) invokes the skill in-process and prints the Started event with a program_id.
2. `mneme run programs.list` against a substrate with at least one program in its index prints the list as a JSON array.
3. `mneme run forecast.bogus` prints an error referencing the available methods and exits 1.
4. `mneme run forecast.update --params '{"program_id":"Q-001","new_evidence":"...","allowed_tools":["WebSearch"]}'` correctly threads the structured args (verified by inspecting the resulting program's manifest.inputs).
5. `cargo build --bin mneme` succeeds; the binary is < 30MB stripped.

## Completion

Implementor runs the four acceptance scenarios manually (the in-process invocation requires a real Anthropic API key for forecast.update); all pass; status flips to `Complete` in the same commit as the code.
