---
id: MNEME-8
title: "`mneme` harness binary"
status: Pending
type: implementation
blocked_by: [MNEME-7]
unlocks: []
confidence: high
---

## Problem

Layers 0–2 are reachable only via Plexus RPC (synapse, custom client). To exercise the harness from a shell, ad-hoc, or as part of a pipeline, the user needs a binary entry point that takes a skill invocation, runs it, and prints the artifact path + structured result.

## Context

- All four layers are functional after MNEME-7. What's missing is the user-facing veneer.
- Synapse already provides a generic schema-driven CLI for any Plexus backend. The harness binary doesn't replace synapse; it bundles substrate + skills into a single process so users don't have to start the substrate separately.
- The binary's primary mode is one-shot: spawn substrate in-process, run one skill invocation, print result, exit.

## Evidence

A bundled binary is the right phase-1 packaging because:

1. Users want one command, not "start substrate, then synapse-call into it." The dev loop must be tight to validate the architecture.
2. Programs are filesystem artifacts; the binary just needs to print the artifact path so the user can `cat` / `ls` it themselves.
3. The substrate has well-defined `with_websocket` / `with_stdio` builders — embedding it is a few lines.

A long-running daemon mode is out of scope; it would invert the program model (currently each invocation = a new program). Daemon mode is post-MVP.

Confidence is `high` because this ticket is clap-args + thin wrapper around already-tested layers. The risk is mostly cosmetic (UX, error messages) which is easy to iterate on after first ship.

## Required behavior

```
mneme run <skill.method> [args...]
mneme list                        # list registered skills with method signatures
mneme inspect <program_id>        # cat manifest.json + artifact.json prettily
mneme replay <program_id>         # post-MVP placeholder; emit "not yet implemented"
```

`mneme run forecast.update --program-id Q-001 --new-evidence "BTC at $95k"`:

1. Parse args; map `<skill>.<method>` to a registered Plexus method.
2. Spin up an in-process substrate with the skill activations + claudecode + swarm registered.
3. Generate a fresh `program_id` (or use one passed via `--program-id`).
4. Open a recorder for the program.
5. Invoke the method with the parsed args.
6. Stream events to stderr (so stdout stays clean for the artifact).
7. On Completed: print `program_id`, `artifact_path`, and pretty-printed `artifact.json` to stdout.
8. On Err: print error to stderr, exit non-zero.

`mneme list` enumerates the registered activations (using `DynamicHub::list_activations_info` from plexus-core) and prints a synapse-style help table.

`mneme inspect <program_id>` reads `programs/<id>/manifest.json` and `programs/<id>/artifact.json` and pretty-prints them.

## Risks

| Risk | Mitigation |
|------|-----------|
| Argument parsing is awkward for nested object args (e.g., a `--prior` that's `{p: 0.3, summary: "..."}`) | Accept `--prior-json '{...}'` for object args; document shorthand `--prior-p 0.3 --prior-summary "..."` for common shapes |
| Substrate startup adds noticeable cold-start latency to every invocation | Acceptable for MVP; daemon mode is the long-term answer |
| Crash leaves a half-written program directory | Recorder writes are atomic per file; `manifest.status: running` indicates an interrupted program |

## What must NOT change

- The substrate, swarm, recorder, or any skill activation. This ticket is integration only.
- The program directory layout.

## Acceptance criteria

1. `mneme/src/bin/mneme.rs` exists and compiles.
2. `mneme run forecast.update --program-id test --new-evidence ""` (against a previously-created forecast) returns exit 0, prints a program_id and an artifact JSON.
3. `mneme list` prints a table including at minimum `forecast`, `swarm` (and any skills landed by phase 2).
4. `mneme inspect <program_id>` against a real program prints both files prettily; against a non-existent id prints a clear error and exits non-zero.
5. End-to-end test (shell script in `tests/e2e_harness.sh` or equivalent in Rust): `mneme run forecast.create ... && mneme run forecast.update ... && mneme inspect <id>`. All three succeed; the inspect output contains the expected fields.
6. `cargo build --release` succeeds; the binary is < 30MB stripped.
7. `cargo build` and `cargo test` pass green for the `mneme` crate.

## Completion

Implementor runs the e2e shell test against a real Anthropic API key; it passes; status flips to `Complete` in the same commit as the code.
