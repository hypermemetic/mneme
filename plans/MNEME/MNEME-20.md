---
id: MNEME-20
title: "claudecode activation modernization: macro visibility, RawRequestContext, cookies + HTTP forwarding, error unification, loopback cleanup"
status: Pending
type: implementation
blocked_by: []
unlocks: []
confidence: medium
---

## Problem

The `claudecode` activation was the first one built in the substrate. Six months and several substrate-level capabilities later, it shows architectural patterns that newer activations (orcha, lattice, cone, solar, health) have refined or replaced. The user's directive: *"please take full advantage of the new functionality. including cookies and http request forwarding."*

A subagent review (recorded in `mneme/plans/MNEME/results/claudecode-review-2026-04-26.md`, to be added) identified seven specific divergences. This ticket bundles the load-bearing ones into one coherent modernization pass.

## Context

Substrate capabilities claudecode is currently bypassing or under-using:

1. **Macro visibility.** Many `#[plexus_macros::method]`-decorated methods (`fork`, `chat_async`, `poll`, `get`, `delete`, `streams`, `get_tree`, `render_context`, `sessions_*`) are private by default. In-process callers (mneme's swarm runtime) had to add `pub` one-by-one. Newer activations declare all wire methods inside the macro block uniformly.

2. **`RawRequestContext` propagation.** Newer activations (`solar/celestial.rs:243`, `health/activation.rs:133`) accept `_raw_ctx: Option<&plexus_core::request::RawRequestContext>` in their dispatch, giving them transparent access to HTTP headers, cookies, peer address, and auth context. claudecode does not use this; everything that needs request-level metadata reinvents it (e.g., `claudecode_loopback.permit()` smuggles session id via query params).

3. **Cookies and HTTP request forwarding.** `RawRequestContext::headers` is a `http::HeaderMap`; `plexus_core::request::parse_cookie` exists. There's a `PlexusRequest` derive that lets activations declare typed extracted views (`#[from_cookie("access_token")]`, `#[from_header("origin")]`, etc.). claudecode does none of this. Use cases that depend on it:
   - **Multi-tenant isolation:** session is bound to the auth-token cookie; only that user can chat with it.
   - **Audit:** every claudecode operation records the originating user/origin from the auth context.
   - **Permission gating:** certain models or tools only available based on auth context.
   - **Outbound forwarding:** when the spawned `claude` CLI makes upstream HTTP requests, it can forward cookies / auth headers from the inbound request.

4. **Error unification.** claudecode yields per-method error enums (`CreateResult::Err`, `DeleteResult::Err`, `GetResult::Err`, ...). Newer activations stream a single `*Event::Error` variant that composes more cleanly when chained.

5. **Loopback architecture.** `claudecode_loopback` is its own activation; session correlation flows via `loopback_session_id` query params and `PLEXUS_SESSION_ID` env vars (`executor.rs:207-322`). Once #2 lands (RawRequestContext access), the loopback can naturally read session id from request context; the env-var smuggling becomes unnecessary.

6. **Executor.** `executor.rs` shells out to `bash → claude` (lines 128-169). Speculative: with `rmcp` now in the substrate's deps, could a more direct invocation skip the CLI? Open question; out of scope for this ticket without an `rmcp`-vs-CLI spike.

## Evidence

The user explicitly named cookies + HTTP forwarding. That maps directly to items #2 + #3. The other items came up in the review and are worth doing in the same pass because they touch the same call sites.

The substrate has accumulated this functionality but claudecode predates it. This is a "bring the oldest activation up to current standards" pass, not a redesign.

## Required behavior

### Phase A: macro visibility (small, mechanical)

Move every `#[plexus_macros::method]` method inside the `#[plexus_macros::activation]` impl block. Drop manual `pub` declarations. The macro generates appropriate visibility. Re-export only what consumers actually need from `mod.rs` (currently, in-process callers reach `fork`/`get` directly; this stays).

### Phase B: RawRequestContext propagation (small-medium)

Extend `claudecode.create` and `claudecode.chat` to receive `Option<&RawRequestContext>` from the macro-generated dispatch. Store it in the `ClaudeCodeConfig` for the session, so subsequent `chat` calls can access the original creator's context.

The macro change required: `#[plexus_macros::method]` should optionally accept a `_raw_ctx: Option<&RawRequestContext>` parameter and inject it from dispatch. This may be already supported (check newer activations); if not, may require a small `plexus-macros` upstream change (out of scope).

### Phase C: cookies + headers (medium feature)

Add explicit parameters to `claudecode.create`:

```rust
pub async fn create(
    &self,
    name: String,
    working_dir: String,
    model: Model,
    system_prompt: Option<String>,
    loopback_enabled: Option<bool>,
    loopback_session_id: Option<String>,
    forward_cookies: Option<HashMap<String, String>>,    // NEW
    forward_headers: Option<HashMap<String, String>>,    // NEW
) -> ...
```

When `forward_cookies` / `forward_headers` are present, store on the session and inject as env vars (`CLAUDE_FORWARD_COOKIES_JSON`, `CLAUDE_FORWARD_HEADERS_JSON`) in the launched `claude` process. The CLI reads these and adds them to outbound HTTP requests.

After Phase B lands: callers who don't pass these explicitly get them auto-extracted from `RawRequestContext.headers` (cookie parsing via `plexus_core::request::parse_cookie`). Manual params are an override / programmatic-use path.

### Phase D: error unification (medium refactor)

Collapse `CreateResult`, `DeleteResult`, `GetResult`, `ForkResult`, etc. to a single `ClaudeCodeEvent` enum with `Ok`-payload variants and an `Error { stage, message }` variant. Method return types become `impl Stream<Item = ClaudeCodeEvent>`. Old types deprecated; existing callers updated.

### Phase E: loopback cleanup (small, after Phase B)

`claudecode_loopback.permit()` reads session id from the request context (cookie or header) instead of from the query param. The `loopback_session_id` plumbing in `executor.rs:207-322` simplifies to direct context propagation. Optionally: deprecate the `loopback_session_id` parameter on `claudecode.create` once context-based correlation is the default.

## Risks

| Risk | Mitigation |
|------|-----------|
| Breaking existing claudecode callers (orcha, our own forecast skill) | Keep old method signatures; add new params as `Option<...>`; run full test suite; verify orcha integration test |
| Macro upstream change needed for context injection | If `#[plexus_macros::method]` doesn't already inject `_raw_ctx`, this becomes a plexus-macros ticket (out of scope) — Phase B then ships with manual workaround until macro lands |
| Cookie format collisions with existing CLAUDE_* env vars | Pick distinctive names (`CLAUDE_PLEXUS_FORWARD_COOKIES_JSON` etc.) |
| Phases C and E both touch loopback in different ways | Sequence: B → C → E → D |
| Multi-tenant isolation behavior changes existing flows | Default OFF; opt-in via a new flag on `claudecode.create` |

## What must NOT change

- The `claude` CLI invocation contract (exec via bash) — unless the executor spike (separate ticket) replaces it.
- The arbor storage schema or session table layout.
- The `claudecode_loopback` activation's external API surface (changes are internal until Phase E deprecates the query-param approach).

## Acceptance criteria

1. **Phase A:** All `#[plexus_macros::method]` methods declared inside the activation block; no manual `pub` modifiers; `mod.rs` re-exports unchanged from caller perspective.
2. **Phase B:** `claudecode.create` and `chat` receive `RawRequestContext` from dispatch; integration test asserts the context's headers are accessible inside the method body.
3. **Phase C:** `forward_cookies` / `forward_headers` parameters work end-to-end; live test creates a session with `forward_cookies={"session_id":"abc"}`, verifies the env var is set in the spawned `claude` process.
4. **Phase D:** All claudecode methods return `Stream<Item = ClaudeCodeEvent>`; existing tests updated; orcha integration test still passes.
5. **Phase E:** Loopback reads session id from RawRequestContext; query-param path remains as fallback for backwards-compat; integration test exercises the new path.
6. `cargo build --bin mneme-substrate` succeeds; `cargo test --lib` green.

## Completion

All five phases land in separate commits (small-incremental shipping); status flips to `Complete` when Phase E lands and the orcha integration test still passes.

## Open subticket: executor SDK invocation (deferred)

Whether to replace the `bash → claude` CLI invocation with direct `rmcp` or `anthropic` SDK calls is a separate spike (out of scope for this ticket). File as a follow-up if Phase D lands and someone wants to pursue it.

## Reference

Subagent review captured at: (will create) `plans/MNEME/results/claudecode-review-2026-04-26.md`.
