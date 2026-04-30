---
id: MNEME-36
title: "Capacity-aware auto-firing scheduler — drain remaining subscription tokens before each reset"
status: Ready
type: implementation
blocked_by: []
unlocks: []
confidence: medium
severity: Medium
forecast:
  hypothesis: "Will MNEME-36 ship within 14 days AND, in steady-state operation, fire ≥80% as many forecasts per 24h as a fully-saturated continuous run would (measured by tokens consumed via claudecode_messages content vs the substrate's nominal subscription per-day envelope)?"
  resolution_method: "After MNEME-36 ships, run for ≥7 consecutive days. Measure (a) total tokens consumed across forecast.update programs from claudecode.db (sum of message char-counts/4 as a proxy until token capture is wired) and (b) the apparent subscription envelope (max consumed in any rolling 5-hour window where the next forecast.update triggered a 429). Resolves YES if the 7-day token consumption is ≥80% of (max-window × n_windows) with no extended (>4h) idle gaps; NO otherwise; N/A if MNEME-36 isn't built within deadline."
  deadline: "2026-05-14T18:00:00Z"
---

## Problem

The user has a Claude subscription with a per-window token cap that
resets on a rolling 5-hour basis. Tokens unused in a window are
**lost** — there's no rollover. Today the substrate fires forecasts
only when explicitly invoked (via marketwatch cron, ticket forecasts,
or manual forecast.update calls). That leaves significant capacity
on the table during long idle stretches.

The user's stated goal: "set something up to always start firing
predictions with my remaining tokens when we close in on one hour
before token refresh."

The substrate already has the infrastructure for this — what's
missing is a scheduler that:

1. Knows what's worth firing (a backlog of prediction-targets).
2. Knows roughly where we are in the current 5-hour window.
3. Fires forecasts at a rate that drains the window without hitting
   the hard cap.
4. Backs off gracefully when 429s appear.

## Context

**Why this matters:**

The live marketplace pipeline (MNEME-30) is the only data path
that's structurally clean for calibration. Each Manifold market we
forecast becomes a paired (predicted, manifold_p, eventual_actual)
row. After 30 resolutions we can tell whether mneme has a real edge
on out-of-sample crowd; after 100, the noise band tightens. **Time
is the bottleneck — markets resolve over weeks.** The faster we
accumulate forecasts, the sooner we have defensible numbers.

The user is paying for the subscription regardless. If we don't
fire, we're wasting it.

**What's available today:**

- `scripts/marketwatch_live.py` (MNEME-30) — picks open Manifold
  markets, fires forecasts, records pairings. Idempotent within
  thresholds, safe to run repeatedly.
- `scripts/marketwatch_resolve.py` (MNEME-33) — two-phase, survives
  substrate downtime.
- `scripts/inspect_program_tree.py` (MNEME-35) — audit any forecast.

What's missing: a controller that decides *when and how much* to
fire.

**Why not just cron `marketwatch_live` every 30 min:**

A naive fixed-cadence cron has two problems:
1. If we cron at high frequency (every 30 min), we hammer the
   substrate continuously and probably hit 429s during peak windows
   — wasting calls.
2. If we cron at low frequency (every 4 hours, the README's current
   suggestion), we leave huge stretches of capacity unused.

The right approach is a feedback-loop scheduler: fire, observe rate
of success/failure, adjust the next fire-rate.

## Required behavior

### 1. The fire decision

A controller (`scripts/auto_forecast_daemon.py` or a substrate-side
activation) that runs continuously OR on a frequent cron (every 5-10
min). On each tick:

- Pulls a small number of candidate forecasts from a backlog
  (priority: live markets needing refresh > newly-opened markets >
  ticket re-forecasts > backlog ForecastBench questions).
- Decides how many to fire this tick based on the current window
  state (see #2).
- Fires them, sequentially or with low concurrency.
- Records the result; updates the window-state estimate.

### 2. Window-state estimation

We don't have direct API access to "remaining tokens this window."
Estimate it indirectly:

- Track `(timestamp, success_or_429)` for every fired forecast.
- The first 429 in a sequence marks the end of the burnable
  envelope; the time-of-first-429 → time-of-first-success-after
  delta marks the reset window length.
- After observing one full reset cycle, the controller has a model
  of: "we have N successful fires per window before we hit 429."
- Subsequent windows: fire conservatively at first, ramp up if no
  429s appear, slow down as cumulative-since-window-start
  approaches the historical N.

Approximate, not precise. The point is to use ~80% of capacity
without hitting the hard rate-limit cliff (which has cooldowns).

### 3. Last-hour drain

The user's specific framing: "fire predictions when we close in on
one hour before token refresh." This is a special case of #2 —
when the controller estimates we're 1h from a reset and have unused
capacity, it ramps up firing rate to drain remaining headroom.

### 4. Backlog sources

In priority order:

1. **Manifold markets needing refresh** — price moved >5% OR last
   forecast >24h old. (Existing logic in marketwatch_live's
   `needs_forecast`.)
2. **Newly-opened binary markets** — fetch fresh from Manifold.
3. **ForecastBench round-prep** — when a new round drops (every
   other Monday at 00:00 UTC), fire forecasts on its 500 questions
   over the next ~24h. (Cooperates with `forecastbench_submit.py`.)
4. **Ticket re-forecasts** — re-fire predictions on `Ready` tickets
   whose forecast deadline is in the next 7 days, to track drift.

### 5. Safety / graceful degradation

- 429 handling: pause for 15 min on first 429, double the pause on
  consecutive 429s up to 1h.
- Substrate-down: if `synapse substrate hash` fails, sleep 5 min
  and retry; don't crash.
- Disk full: skip firing if `programs/` is >50GB (write a warning
  to a log path the user checks).
- Hard cap on per-day fires: configurable, default 200 (prevents
  runaway loops from burning the whole subscription on a bug).

## Acceptance criteria

1. `scripts/auto_forecast_daemon.py` (or equivalent) runs as a
   long-lived process or via cron-every-10min and fires forecasts
   automatically.
2. State is persisted under `programs/_auto_forecast/` so restarts
   pick up where they left off (window-model, last-fire log, etc).
3. After 24h of continuous operation, the substrate has fired at
   least 50 new forecasts (mostly via marketwatch refresh).
4. After the first reset-cycle observation, the controller's
   per-window cap estimate is within 25% of the actual observed
   max successful fires before 429.
5. README documents the scheduler under a new "Auto-firing"
   section.
6. Tests: scheduler logic (window-state model, backoff curve)
   covered by unit tests with a mock 429-emitter.

## Out of scope

- A control panel UI for the user to see live state. (Useful
  later; v1 logs to disk + the inspect-tree tooling can read it.)
- Cross-substrate scheduling (only one substrate per machine).
- Token-cost-of-each-forecast prediction (we don't have token
  telemetry yet on the OAuth path; window-state estimation is
  empirical not predictive).

## Risks

| Risk | Mitigation |
|------|-----------|
| 429-detection unreliable; substrate masks the underlying error | Add explicit error-string parsing in claudecode chat handler; if "rate" or "limit" or "429" appears, mark window as exhausted |
| User has multiple Anthropic-using tools; substrate's fires interact with their other usage in the same subscription | Configurable hard-cap; log so user can correlate with their other tools' usage if it surfaces |
| Manifold API rate-limits us before Anthropic does | The manifold path's existing 429-handling (MNEME-33's deletion grace) already covers this |

## Why not a substrate activation

Could be one (`scheduler` as a Plexus RPC namespace). Reasons to
keep it client-side as a Python daemon for v1:

- Fast to ship; no Rust changes.
- Easy to stop/restart without bouncing the substrate.
- Logs are plain files; debugging is grep + tail.
- If demand emerges for non-Python consumers, port to substrate-
  side later.

## Completion

Tests + 24h observation + scheduler clearly draining capacity
without hitting the hard rate-limit wall. Status → Complete.
