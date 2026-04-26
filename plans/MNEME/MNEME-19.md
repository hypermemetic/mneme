---
id: MNEME-19
title: "Synapse array-arg syntax for `--allowed-tools` and other Vec params"
status: Pending
type: spike
blocked_by: []
unlocks: []
confidence: n/a
---

## Question

What is the correct synapse CLI syntax for passing an array-typed parameter (`Vec<String>`) to a Plexus method, and does the current synapse build support it cleanly?

## Context

During live testing of `forecast.update --allowed-tools '["WebSearch"]'`, synapse passed the literal JSON string `"[\"WebSearch\"]"` as a single Vec<String> element rather than parsing the bracketed value as a JSON array. Result: the trial saw a tool named `["WebSearch"]` (the string), which doesn't match any real tool name, so trials effectively had no tools — silently. The forecast still produced output (Claude reasoned from training data), so the bug wasn't loud.

This is the second time array-arg parsing has come up: the same shape applies to `disallowed_tools`, future skills' `Vec<Finding>` outputs, etc.

## Setup

1. Read synapse's CLI parsing code (Haskell — likely in `synapse/src/Synapse/Cli/` or similar).
2. Test the following candidates against a known Vec<String> param:
   - `--allowed-tools '["WebSearch","Read"]'` (current attempt; doesn't work)
   - `--allowed-tools WebSearch --allowed-tools Read` (repeated-flag pattern)
   - `--allowed-tools WebSearch,Read` (comma-separated)
   - `--params '{"allowed_tools":["WebSearch","Read"]}'` (full JSON params escape hatch)
3. For each candidate, inspect the resulting `programs/<id>/manifest.json` `inputs.allowed_tools` field to see what got through.

## Pass condition

At least one of the candidate syntaxes (a) works without changes to synapse and (b) is documentable as the recommended pattern for array args. The `--params '{...}'` escape hatch counts as a pass even if the specific flag syntax doesn't, since it's reliable.

## Evidence to record

- Which syntax(es) work and which don't.
- The resulting `inputs.allowed_tools` shape on disk for each.
- Whether synapse's IR/schema sees the param as `array` or as something else (`-s` flag emits the schema).
- If a fix is needed in synapse, what file/function would change.

## Fail → next

If no flag syntax works: document `--params '{...}'` as the canonical way to pass arrays, add an example to the mneme README, and accept the UX gap.

## Fail → fallback

Same as next — `--params` always works.
