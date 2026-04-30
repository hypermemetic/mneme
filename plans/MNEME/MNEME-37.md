---
id: MNEME-37
title: "Substrate error-class detection — auth, rate-limit, and empty-response failures shouldn't masquerade as 'no fenced JSON block'"
status: Ready
type: implementation
blocked_by: []
unlocks: []
confidence: high
severity: High
forecast:
  hypothesis: "Will MNEME-37 ship within 7 days AND, on a synthetic test where the OAuth token is intentionally expired before firing 5 forecasts, every program close with an explicit `kind: AuthExpired` error AND the controlling script (forecastbench_submit.py / forecastbench_holdout_run.py / marketwatch_live.py) abort with a clear `OAuth token appears expired — run claude /login and bounce the container` message instead of recording 5 silent `AllTrialsFailed` failures?"
  resolution_method: "Manually invalidate the substrate's CLAUDE_CODE_OAUTH_TOKEN (e.g. by editing the env var in the running container or replacing it with garbage). Fire 5 forecasts via forecastbench_submit.py. Resolves YES if (a) all 5 program error.json files contain `kind: AuthExpired` (not AllTrialsFailed), (b) the script logs a clear auth-expired diagnostic and exits with non-zero before continuing to fire more, (c) the run aborts within ~30s of the first failure rather than burning through the whole batch. NO if any criterion fails. N/A if MNEME-37 isn't built within 7 days."
  deadline: "2026-05-07T18:00:00Z"
---

## Problem

Today's probe (50 questions on the 2026-04-26 round) failed at ~65%
with a confusing class of error: `parse step 0: no fenced JSON step
block found in LLM output`. Investigation revealed the trial's
buffered "assistant" content was actually the API error string:

```
Failed to authenticate. API Error: 401 {"type":"error","error":
{"type":"authentication_error","message":"Invalid authentication
credentials"},"request_id":"..."}
```

The substrate happily passed this through `chat → buffer → parse_step`,
where it failed parser validation as "no fenced JSON block" — three
levels removed from the actual cause. The user got `AllTrialsFailed`
in `error.json` and had no signal that auth was the problem.

Same failure shape would happen with rate limits (429), service
unavailability, and certain malformed completion events — anything
where the Claude Code SDK passes an error string through the content
stream instead of via the proper error channel.

This is a real reliability gap. We can lose hours and money
forecasting "into the void" without the substrate even noticing.

## Required behavior

### 1. Detect known infrastructure error patterns at the trial layer

In `mneme/runtime/claudecode_swarm_runtime.rs::run_one_trial` and
`run_iterative_trial` (and `ClaudecodeStepDriver::next_step`), after
buffering the chat response, scan for known error-class fingerprints
BEFORE passing to `parse_step`. Detected classes:

| Class | Fingerprint patterns | Trial outcome |
|---|---|---|
| Auth | `"authentication_error"` OR (`"401"` AND `"authentication"`) OR `"Failed to authenticate"` OR `"Invalid authentication credentials"` | `Err("AuthExpired: <first line of detected message>")` |
| Rate limit | `"rate_limit_error"` OR (`"429"` AND `"rate"`) OR `"rate_limit_exceeded"` | `Err("RateLimited: <first line>")` |
| Service | `"service_unavailable"` OR `"5xx"`-shaped patterns | `Err("ServiceError: ...")` |
| Empty response | `buffer.is_empty()` after `Complete` event | `Err("EmptyResponse")` (already exists in some paths; standardize) |

Pattern matching is conservative: case-insensitive, scans the first
~2000 chars of buffered content. Real assistant text that happens to
contain the substring `401` (e.g. apartment 401) won't match because
the AND with "authentication" is required.

### 2. Surface error classes up to the activation level

`SwarmError::AllTrialsFailed` is too coarse. Add a new variant:

```rust
pub enum SwarmError {
    /// All K trials hit the SAME infrastructure error class
    /// (auth/rate-limit/service). The user needs to fix
    /// infrastructure, not retry.
    AllTrialsInfraFailed { class: String, first_message: String },
    /// All K trials failed but for different / mixed reasons.
    AllTrialsFailed,
    // ... existing variants
}
```

When all trial outcomes match the same infra class (auth on every
trial, or rate-limit on every trial), elevate to
`AllTrialsInfraFailed`. This signal is what scripts use to decide
"abort the run" vs "record one failure and continue."

### 3. Propagate through `forecast.update`'s background task

`run_update_in_background` should write a structured error on the
program when it sees `AllTrialsInfraFailed`:

```json
{
  "kind": "AuthExpired",       // or "RateLimited", "ServiceError"
  "message": "OAuth token appears expired",
  "stage": "swarm.trial",
  "first_observed_message": "Failed to authenticate. API Error: 401 ..."
}
```

The `kind` field is the actionable signal — scripts read it.

### 4. Operator scripts abort on auth/service errors

`forecastbench_submit.py`, `forecastbench_holdout_run.py`, and
`marketwatch_live.py` already aggregate failures. After this ticket,
they additionally:

- Inspect each program's `error.json` (when it exists)
- If `kind` is `AuthExpired` or `ServiceError`, exit with non-zero
  and a clear message:

  ```
  ERROR: substrate forecasts failing with AuthExpired.
  Probable cause: OAuth token expired.
  Fix: on the host, run `claude /login`, then `bash scripts/run_container.sh up -d`.
  Aborting (consumed: 1 program, 0 successes).
  ```

- For `RateLimited`, sleep 15 min and retry (existing behavior in
  forecastbench_submit's 429 detector becomes cleaner with explicit
  signal).

### 5. Substrate startup probe (defense in depth)

Optional v1.5: before `forecast.update` returns its `Started` event,
make a one-message "ping" call. If it returns an auth/service
error, the program closes immediately as `AuthExpired` without ever
fanning out K trials. Saves the K-trial-cost of finding out.

Cost: one extra chat call per forecast.update. ~5% overhead per
forecast. Worth it if it saves us from burning 50 forecasts during
an auth outage.

## Acceptance criteria

1. Pattern-detection helpers in
   `mneme/runtime/claudecode_swarm_runtime.rs` (or a new
   `infra_error.rs`) covering auth, rate-limit, service, empty
   classes; pure functions; ≥6 unit tests covering each pattern
   plus false-positive guards.
2. `SwarmError::AllTrialsInfraFailed { class, first_message }`
   variant exists and is returned when all K trials match the same
   class.
3. `run_update_in_background` writes `kind` into `error.json`'s
   structured payload.
4. `forecastbench_submit.py` aborts on `AuthExpired` /
   `ServiceError` with a clear human-readable diagnostic.
5. `marketwatch_live.py` aborts on `AuthExpired`; sleeps + retries
   on `RateLimited`.
6. README troubleshooting section gains an entry: "All forecasts
   failing with AuthExpired? → run `claude /login`, then bounce the
   container."
7. `cargo test --lib` green.
8. Manual scenario test: synthetic auth invalidation + 5-forecast
   probe behaves per the forecast hypothesis.

## Why High severity

Today we lost ~5 minutes and an unknown amount of subscription burn
on auth-failed calls without realizing it. Worse: the failure mode
*looks like* a model-quality bug (parse failures) and could have
sent us down a rabbit hole investigating cleanup-recovery / prompt
issues instead of the actual cause.

A future automated firing daemon (MNEME-36) would compound this:
the daemon would happily retry forever, eating the whole
subscription, with all programs writing "AllTrialsFailed" until
someone noticed.

## Out of scope

- Auto-refreshing the OAuth token from inside the substrate (the
  Keychain integration in `run_container.sh` already handles refresh
  on container restart; auto-refresh during runtime needs a
  separate ticket if we ever want it).
- Detecting model-side errors that present as confused but parseable
  output (e.g. the model says "I cannot help with that" — that's a
  prompt/policy issue, not infrastructure; existing parser-failure
  paths handle the consequence).
- Per-call retry with exponential backoff (the user explicitly
  prefers explicit-abort over silent-retry for clarity).

## Completion

Tests pass + manual auth-invalidation scenario works as described
in the forecast hypothesis. Status → Complete.
