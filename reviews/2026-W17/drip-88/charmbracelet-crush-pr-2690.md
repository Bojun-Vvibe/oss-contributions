---
pr: 2690
repo: charmbracelet/crush
sha: 7ff330a285702e3e2fbefe33b62325e899018082
verdict: merge-after-nits
date: 2026-04-27
---

# charmbracelet/crush #2690 — fix(db): prevent SQLITE_NOTADB corruption under concurrent sub-agents

- **Author**: (Opus 4.7 via crush, per PR body)
- **Head SHA**: 7ff330a285702e3e2fbefe33b62325e899018082
- **Size**: +15/-1 across `internal/db/connect.go`, `internal/db/connect_modernc.go`, `internal/db/connect_ncruces.go`. Closes #2682.

## Scope

Three coordinated changes attacking the same `SQLITE_NOTADB (26)` corruption under parallel sub-agents:
1. `connect.go`: `db.SetMaxOpenConns(1)` to serialize all access through one connection.
2. `connect_modernc.go`: add `_txlock=immediate` so writers acquire RESERVED lock up front (avoid deferred→writer upgrade contention).
3. `connect_ncruces.go`: same `_txlock=immediate` via DSN string (the ncruces driver requires the `file:` prefix to parse query params).

This **overlaps with PR #2691** (already reviewed in W17 drip-87), which sets the same `SetMaxOpenConns(1)` cap with a longer block-comment diagnosis. The two PRs need to be reconciled before either merges.

## Specific findings

- `internal/db/connect.go:48-50` — `db.SetMaxOpenConns(1)` is the conservative, correct fix for the symptom. SQLite serializes file-level writes anyway, so a 1-conn pool eliminates the WAL-frame-interleaving race without losing real concurrency. The 5-line block comment explains the *why* (concurrent sub-agents → multiple handles → interleaved WAL frames + auto-checkpoint races → main DB header vs WAL desync on cancel/kill). This is the same fix as #2691; the comment is shorter here.
- `internal/db/connect_modernc.go:18-20` — `params.Set("_txlock", "immediate")` makes every `BEGIN` acquire a RESERVED lock immediately rather than upgrading from SHARED→RESERVED at first write. This is the right call **even with `SetMaxOpenConns(1)`**: it prevents `database is locked` errors when a transaction blocks waiting to upgrade, which is a separate pathology from the WAL-desync corruption. Two tools, two distinct problems — both worth fixing.
- `internal/db/connect_ncruces.go:14-17` — the ncruces driver path. The DSN-string approach (`fmt.Sprintf("file:%s?_txlock=immediate", dbPath)`) is required because the ncruces driver doesn't expose a Go-level config struct for `_txlock`. The `file:` prefix is mandatory for query-param parsing in this driver — author correctly documented that in the comment. Verify `dbPath` cannot already contain `?` or be already a `file:` URI; otherwise this `Sprintf` will produce malformed DSN. A `strings.Contains(dbPath, "?")` guard or `url.URL` construction would be safer.
- **Diagnosis quality**: the PR body's RCA is good (multi-handle WAL writers desyncing, mid-checkpoint kill leaves header/WAL desync). The fix combines "fewer handles" + "fail fast on writer-upgrade contention", which addresses both the causal mechanism and the secondary failure mode. This is actually *better-shaped* than #2691 because #2691 only does the conn cap — `_txlock=immediate` here is a meaningful additional improvement.
- **Missing**: no test. SQLite corruption races are notoriously hard to reproduce in unit tests, but a stress test that spawns N goroutines doing parallel write transactions for K seconds and asserts the DB still passes `PRAGMA integrity_check` after a forced cancellation would substantially raise confidence. Even a test that just exercises `_txlock=immediate` is acquired (via `BEGIN IMMEDIATE` showing up in trace logs) would help.

## Risk

Medium-low. `SetMaxOpenConns(1)` will serialize all DB access — for crush's actual workload (one user, a handful of concurrent sub-agent DB writes) this should be invisible; for any future use case that wants concurrent reads while writing, the cap will become a bottleneck. `_txlock=immediate` may surface `SQLITE_BUSY` errors that were previously masked by deferred-to-writer upgrade — make sure the surrounding code retries `BUSY` correctly (likely already does via the SQLite busy-handler PRAGMA, but worth a `git grep` for `SQLITE_BUSY` handling).

## Verdict

**merge-after-nits** — (1) reconcile with PR #2691 (this one is the better fix; close #2691 in favor of this if maintainers agree), (2) add the DSN-safety guard in `connect_ncruces.go` for `dbPath` already containing `?`, (3) ship at least a smoke test that the `_txlock=immediate` is being applied. The combination of conn-cap + immediate-lock is more correct than either alone.
