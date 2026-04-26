---
id: BLFX-4
title: "Iterative agent loop (per-trial multi-step belief update)"
status: Pending
type: implementation
blocked_by: [BLFX-2, BLFX-3, BLFX-S02, BLFX-S04]
unlocks: [BLFX-13]
confidence: low
---

## Problem

This is the core architectural change. Our current `swarm.trial` runs each trial as a single `claudecode.chat` call. The paper's BLF runs each trial as an iterative loop: at each step the LLM picks an action AND updates its belief; the action is executed; the result is appended to history; repeat up to T_max=10 steps or until the LLM submits.

The paper's ablation: replacing this iterative loop with a single batch-search-then-reason approach degrades Brier Index by 3.8. This is the **single biggest non-search algorithm change** between our implementation and the paper.

## Context

After BLFX-2 (structured belief state) and BLFX-3 (Action + parser), this ticket assembles them into the loop in `mneme-substrate/src/mneme/runtime/claudecode_swarm_runtime.rs::run_one_trial`.

The current `run_one_trial`:
- Forks a session
- Sends one chat
- Drains events
- Returns the final assistant text

The new `run_one_trial`:
- Forks a session
- For t = 1..T_max:
  - Sends a chat (with current history)
  - Parses response into `(Action, ForecastState)` via `parse_action`
  - If `Action::Submit { probability }`: terminate, return belief state
  - Otherwise: execute action, get `Observation`, append `(action, observation, belief)` to history, continue
- If T_max hit without submit: force-submit using the most recent belief

## Evidence

Per the paper (§§1, 3): *"the structured belief state is almost as impactful as web search access, and shrinkage aggregation and hierarchical calibration each provide significant additional gains."* Removing the iterative loop replaces structured-belief + search with batch-search; that's a -3.8 BI hit, comparable to removing search entirely (-4.6).

The risk is real because:
- Each trial now makes ~5–10 LLM calls instead of 1. Cost goes up ~5–10x. (BLFX-S03 estimates this.)
- Loop complexity introduces failure modes: parse errors mid-loop, action execution failures, infinite-loop possibilities.
- The "force submit at T_max" logic needs to be robust — the model may not have a good probability after T_max iterations.

The reward, if it works, is the algorithm the paper actually validates, not an approximation.

## Required behavior

`run_one_trial` becomes:

```rust
async fn run_one_trial<P: HubContext + 'static>(
    claudecode: Arc<ClaudeCode<P>>,
    parent: String,
    new_name: String,
    initial_question: String,
    cutoff_date: Option<DateTime<Utc>>,
    max_steps: u8,                                           // T_max, default 10
    timeout: Duration,
    allowed_tools: Option<Vec<String>>,                      // claudecode tool subset
) -> Result<ForecastState, String> {
    // Fork
    // Initialize belief b_0 with p=0.5, empty evidence
    // For t = 1..=max_steps:
    //   1. Compose user prompt: "current history is X; pick next action and update belief"
    //   2. Send chat
    //   3. Parse response → (action, belief)
    //   4. If action == Submit(p): return belief with p
    //   5. Else: execute action, get observation
    //   6. Append (action, observation, belief) to history
    // Force submit: return last belief
}
```

| Scenario | Expected outcome |
|----------|-----------------|
| Trial submits within T_max | Returns the submitted belief state |
| Trial hits T_max without submit | Force-submit using most recent belief; mark belief.confidence as low |
| Parse error mid-loop | Append error to history as assistant message + corrective user prompt; retry once; if still failing, abort trial as failure |
| Action execution times out | Observation::Error, loop continues |
| Trial timeout (overall) | Abort trial; return as TrialFailure |

## Risks

| Risk | Mitigation |
|------|-----------|
| Cost per trial increases ~5–10x | BLFX-S03 quantifies; if unaffordable, run smaller benchmarks or use Sonnet |
| LLM forgets to update belief, just emits action | Parser requires both fields; missing belief = parse error = retry |
| Loop infinite-recurses on a single action (e.g., always lookup_url) | T_max cap is the hard stop |
| Forks balloon — every trial creates a new claudecode session | Already handled by attribution; consider cleanup-on-completion |

## What must NOT change

- `swarm.trial`'s external API.
- `forecast.update`'s external API.
- The aggregation step (BLFX-5) consumes whatever this returns.

## Acceptance criteria

1. `run_one_trial` rewritten in `claudecode_swarm_runtime.rs` to implement the loop.
2. Integration test against `DeterministicMockSwarmRuntime`: trial submits at step 3; assert belief returned matches submitted probability.
3. Integration test against mock: trial hits T_max=5 without submit; assert force-submit uses most recent belief and marks confidence as low.
4. Live test: `forecast.update` against a real claudecode session produces an artifact whose trial sessions show multi-step conversation history (verifiable via `programs.inspect` showing >1 chat turn per trial session).
5. The cost-per-question on the live test is recorded (in `programs/_calibration/` or session log).
6. `cargo test --lib` passes green.

## Completion

Tests pass + live verification with cost recorded; status flips to `Complete` in the same commit.
