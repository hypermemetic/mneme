# Open Concerns

Real open concerns about the system. Process learnings live in skills (`presence`, `autonomous-work`); operational issues live in commits and `git log`. Only what's load-bearing across sessions belongs here.

---

## Numerical: Platt fit unstable on extreme probabilities

The current Newton-Raphson Platt scaling implementation can fail to converge when training observations include probabilities outside roughly `[0.1, 0.9]`. Inside that band, fits are reliable. Outside, the iteration can oscillate.

**Why this matters:** if a skill produces high-confidence outputs (`p > 0.9` or `p < 0.1`) and they get fed into calibration, the bias parameters may fail to update.

**Fix path:** Newton with line search, or Levenberg-Marquardt damping, or switch to a published implementation. Deferred until calibration data shows it actually surfaces in real use; the e2e pipeline test uses moderate probabilities and passes.

---

## Architectural: skill activations don't yet hold a runtime handle

Skill activations (`forecast`, `ticketing`, `planning`, `security_review`, `strong_typing`) compile and respond to method calls but they don't currently hold a `SwarmRuntime`, so methods that need orchestration emit `Error { stage: "swarm-not-wired" }` rather than running.

**The pending decision:** how the runtime reaches the activation. Three options:
- Activation holds `Arc<dyn SwarmRuntime>` field (simplest; what the existing `claudecode` pattern would suggest)
- Activation generic over a context type (matches `ClaudeCode<P: HubContext>`'s pattern)
- Runtime accessed via tokio task-local or jsonrpsee `Extensions` (keeps method signatures clean but adds spooky action)

The end-to-end pipeline (program → trial → aggregate → calibrate → artifact) is proven correct in tests against a `DeterministicMockSwarmRuntime`. The wiring decision is downstream; the math and lifecycle are not blocked on it.

---

## Upstream: `plexus-macros` may have a missed deprecation site

A grep for the deprecated `hub_method` / `hub_methods` macros in the broader hypermemetic project turned up one live source file: `plexus-macros/src/codegen/mod.rs`. Likely the macro crate's own deprecation pathway (correct), but not verified.

Out of mneme's lane to fix; flagged for whoever maintains plexus-macros.

---

## Upstream: license disagreement in inherited substrate

Upstream `plexus-substrate/Cargo.toml` declares AGPL-3.0-only; upstream `README.md` says MIT. mneme-substrate inherited both files and the new substrate README points at Cargo.toml as the source of truth. The upstream discrepancy still exists.

Out of mneme's lane to fix.
