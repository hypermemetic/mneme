---
id: MNEME-18
title: "Investigation: programs.list cross-restart correctness in SQLite index"
status: Pending
type: spike
blocked_by: []
unlocks: []
confidence: n/a
---

## Question

Does the `MnemeStorage` SQLite index at `.substrate/mneme-programs.db` persist program rows across substrate restarts as designed, or does something cause earlier rows to be invisible to subsequent `programs.list` calls?

## Context

During live testing on 2026-04-26:
- Run 1: `forecast.update` produced program `1f55f9c2-b8dd-46cc-9603-4c35297be98d` (status=completed).
- Substrate restarted.
- Run 2: `forecast.update` produced program `06bb5c5c-1d33-4fc1-9136-e94fd4fe3710` (status=completed).
- `programs.list` after run 2 returned only `06bb5c5c...` with `count: 1`. The earlier `1f55f9c2...` row was not returned.

Both runs were from the same working directory (`.runtime/`), so they share the same `.substrate/mneme-programs.db` file. The expected behavior: both rows should be returned.

## Setup

1. Reproduce: launch substrate from a fresh working directory; invoke `forecast.update` once; capture the program_id; restart substrate; invoke `forecast.update` again; capture the second program_id; call `programs.list --limit 10`.
2. Independently inspect the SQLite db: `sqlite3 .substrate/mneme-programs.db "SELECT program_id, status, started_at FROM mneme_programs ORDER BY started_at DESC;"` — confirm whether both rows ARE in the database.
3. If both rows are in the DB but `programs.list` returns one: the bug is in the activation's listing logic (likely a stale connection to a previous DB instance, or an incorrect filter).
4. If only one row is in the DB: the bug is in the open path — `Program::open` may be writing to a different DB instance per substrate process, or `MnemeStorage::open` may be truncating.

## Pass condition

Both rows visible via `programs.list` after the restart, AND both rows present in the SQLite db file via direct sqlite3 inspection.

## Evidence to record

- The SQLite db file path used by each substrate run (verify it's the same file).
- The contents of `mneme_programs` after each run via direct sqlite3 query.
- Whether `MnemeStorage::open` runs `CREATE TABLE IF NOT EXISTS` (it does in the current code) or `DROP TABLE` (it should not).
- Connection pool behavior: does sqlx hold any per-process state that would prevent seeing rows written by a previous process?

## Fail → next

If reproduced, diagnose: most likely candidates are
- `.substrate/mneme-programs.db` path is computed CWD-relative differently per run
- File permissions / locking issue
- Pool max_connections=5 with WAL mode interaction
- A stale prepared statement caching old results

## Fail → fallback

If we can't fix the cross-restart issue: document the limit in `programs.list`'s doc comment and have it fall back to scanning `programs/` directories on disk (which always contain manifest.json and are the truth).
