# Issues Log

Append-only journal of issues discovered during work on mneme and adjacent skills. No issue too small to log. Entries are chronological. Resolution can be inline (one-line fix) or pointer to a ticket.

**Schema per entry:**
- **ID:** `LOG-N`, monotonic
- **Surfaced by:** what triggered the discovery (which check, which sweep, which user nudge)
- **Issue:** what's wrong, with file:line where applicable
- **Fix:** what was done; "logged-only" if no action yet
- **Status:** `resolved` | `open` | `deferred`

---

## 2026-04-25

### LOG-1: `MEMORY.md` referenced deprecated `#[hub_methods]` macro
- **Surfaced by:** User flagged `plexus_macros::activation` / `plexus_macros::method` as the current names; old `hub_methods` / `hub_method` deprecated
- **Issue:** `~/.claude/projects/-Users-shmendez-dev-controlflow-hypermemetic/memory/MEMORY.md` line 9 (and elsewhere) referenced `#[hub_methods]` as if current
- **Fix:** Updated memory entry with the new macro names and a pointer to `plexus-substrate/src/activations/claudecode/activation.rs` as canonical
- **Status:** resolved

### LOG-2: `create-plexus-backend` SKILL.md referenced deprecated macros in prose
- **Surfaced by:** Same context as LOG-1 (project-wide grep)
- **Issue:** `skills/skills/create-plexus-backend/SKILL.md` had two prose references to `#[hub_methods]` and `#[plexus_macros::hub_method(streaming)]`
- **Fix:** Rewrote both lines to use `#[plexus_macros::activation]` and `#[plexus_macros::method(streaming, ...)]`
- **Status:** resolved

### LOG-3: `activation.rs.template` used deprecated macros
- **Surfaced by:** Same context as LOG-1
- **Issue:** `skills/skills/create-plexus-backend/templates/activation.rs.template` had `#[plexus_macros::hub_methods(...)]` on the impl block and `#[plexus_macros::hub_method(...)]` on the example method — a stamped-out crate would fail to compile against current `plexus-macros 0.3.10`
- **Fix:** Replaced both with `#[plexus_macros::activation]` / `#[plexus_macros::method]`
- **Status:** resolved (verified by stamping a test crate and running `cargo check` — see LOG-5 verification)

### LOG-4: Method `description = "..."` placed inside the macro instead of as a `///` doc comment
- **Surfaced by:** User said "make sure to use doc strings to annotate the methods"
- **Issue:** `activation.rs.template` placed the method's description as a `description = "..."` arg inside `#[plexus_macros::method(...)]`. Canonical claudecode pattern uses `///` doc comments above the method; the macro carries only `params(...)` and (optionally) `streaming`
- **Fix:** Updated template to use `///` doc comment above `hello`, removed `description` from the macro args, kept `params(message = "...")`. Added explanatory note in the template body and a corresponding bullet in `create-plexus-backend/SKILL.md` "Conventions" section
- **Status:** resolved

### LOG-5: Nine DAG symmetry errors across MNEME tickets
- **Surfaced by:** Self-imposed `blocked_by` ↔ `unlocks` symmetry check (Python script, ad-hoc)
- **Issue:**
  - `MNEME-8.unlocks` was `[]` but 7 downstream tickets (MNEME-9..15) declared `blocked_by: [MNEME-8]` — missing back-pointers on 7 edges
  - `MNEME-3.unlocks` was `[MNEME-4]` but `MNEME-6.blocked_by` includes `MNEME-3` — missing MNEME-6
  - `MNEME-4.blocked_by` was `[MNEME-3, MNEME-S02]` but `MNEME-S03.unlocks` includes `MNEME-4` — missing MNEME-S03
- **Fix:** Added the missing entries to MNEME-3, MNEME-4, MNEME-8 frontmatter. Re-ran symmetry check: 0 errors across all 18 tickets
- **Status:** resolved

### LOG-6: Stamped template generates `unused import` warning
- **Surfaced by:** `cargo check` on the stamped-out test crate (see LOG-3 verification)
- **Issue:** `src/builder.rs` template imports `use crate::activations::*` but the example registration line is commented out, producing a `warning: unused import` on every fresh scaffold. Harmless, but every new Plexus backend will start with a yellow squiggle
- **Fix:** Logged-only. Three options to resolve later: (a) uncomment the example registration so the import is used, (b) add `#[allow(unused_imports)]` to the template's `use` line, (c) accept the warning as a "delete me when you register your first plugin" signal. Recommendation: (a) — pre-register the `Hello` example so the scaffold compiles clean and the user has a working baseline to delete
- **Status:** open (small create-plexus-backend skill cleanup)

### LOG-7: pdfimages extraction is noisy by default
- **Surfaced by:** Background agent extracted 30 PNGs from `docs/blf-paper.pdf`, of which only 3 were actual figures (the rest were per-page renders, decorative elements, or rasterized math)
- **Issue:** Not a bug, but a process note: `pdfimages -png` does not distinguish "figure" from "any embedded raster." Curating extracted images by hand was the right move; the agent's first pass also hit the conversation image-dimension limit when trying to caption all 30 at once
- **Fix:** Logged-only. For future runs: thumbnail to ≤800px before viewing, examine the largest few first (real figures tend to be large), discard <10KB images automatically as decorative
- **Status:** deferred (process improvement; not blocking)

### LOG-8: One live source file in upstream still uses `hub_method`/`hub_methods`
- **Surfaced by:** Project-wide grep for deprecated macros (filtered to live `/src/` files, excluding `.bak`/`.disabled`/`target/`)
- **Issue:** `plexus-macros/src/codegen/mod.rs` references `hub_method` / `hub_methods` — likely the macro crate's own deprecation pathway code (not stale usage), but not verified
- **Fix:** Logged-only — out of mneme's scope, requires plexus-macros maintainer attention. If the deprecation pathway is what's there, this is correct; if it's a missed migration site, the macro crate needs a small change
- **Status:** deferred (upstream concern, not in mneme's lane)
