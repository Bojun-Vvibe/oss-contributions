# charmbracelet/crush#2691 — fix(db): cap SQLite pool to one writer to prevent NOTADB corruption

- **PR:** https://github.com/charmbracelet/crush/pull/2691
- **Head SHA:** `0b82b179a9c64a7075ef0074f82abc26cdcfd089`
- **Files:** 1 — `internal/db/connect.go` (+12/-0).
- **Verdict:** **needs-discussion**

## Context

Refs `charmbracelet/crush#2682`. The `*sql.DB` returned by
`db.Connect` is shared by every service (session, message, history,
filetracker) and every parallel sub-agent. Without an
explicit `SetMaxOpenConns`, Go opens an unbounded number of SQLite
handles. Concurrent WAL frame writes + auto-checkpoints from
multiple handles, combined with mid-checkpoint cancellation
(context cancel, SIGINT, OOM), can desync the main DB file from the
WAL and produce `SQLITE_NOTADB (26)` — `file is not a database` —
on next open. Recovery requires deleting `.crush/crush.db`, which
loses the project's session history.

## Important context: this PR competes with #2690

Crush PR **#2690** (already in INDEX.md, drip-9) addressed exactly
the same `SQLITE_NOTADB` bug with the more comprehensive fix:
`MaxOpenConns(1)` **plus** `_txlock=immediate` in the connection
string. PR #2690 is +15/-1 lines; this PR (#2691) is +12/-0 lines
and **omits** the `_txlock=immediate` half of the fix. Both PRs
target the same upstream issue. Whoever reviews this PR needs to
decide which one to merge — they should not both land.

The relationship matters because `_txlock=immediate` and
`MaxOpenConns(1)` solve overlapping but **not identical** failure
modes:

- `MaxOpenConns(1)` serializes all writes through a single
  connection. This is sufficient against *concurrent-writer*
  WAL-checkpoint interleaving, which is the dominant cause of the
  observed `NOTADB` symptom.
- `_txlock=immediate` upgrades transaction begin from `BEGIN
  DEFERRED` to `BEGIN IMMEDIATE`, which acquires the reserved lock
  at txn-start instead of at first-write. Combined with
  `busy_timeout`, this reduces a separate failure mode where two
  concurrent `BEGIN DEFERRED` transactions both think they can
  upgrade to writer and one fails noisily mid-transaction.

So #2690's belt-and-suspenders is genuinely safer than this PR's
single-mitigation approach. But this PR has its own merit (see
"Strengths" below).

## Design — what changed in this PR

The entire change is a single call inserted between
`sql.Open` (which already happens earlier in `connect.go`) and the
existing `db.PingContext`:

```go
db.SetMaxOpenConns(1)
```

Wrapped in a 10-line code comment that — and this is the strongest
part of the PR — accurately and concisely names the failure mode,
the root cause (parallel sub-agents + Go's default unbounded pool +
WAL checkpointing + cancel-mid-checkpoint), and the rationale for
why `busy_timeout` (already 30s in the connection string) is
sufficient to absorb the contention introduced by serial writes.

That comment is excellent — future readers of `connect.go` will
understand exactly why a one-line `SetMaxOpenConns` exists, which
is more than can be said for most one-liner DB-config changes.

## Strengths

- **Right-sized for the dominant failure mode.** If the project
  data shows that concurrent-handle WAL interleaving is what's
  actually causing the `NOTADB` reports (vs. concurrent-writer
  upgrade races), then `MaxOpenConns(1)` alone is the correct
  minimal fix.
- **The comment is documentation-grade.** It names the bug, the
  symptom, the root cause, the recovery path users had to take,
  and the rationale for why this fix is sufficient. This is the
  kind of comment that survives 5 years of refactoring.
- **Zero behavior change at call sites.** No service needs to know
  about the pool change; `busy_timeout=30s` already in the
  connection string queues concurrent callers behind the single
  writer.
- **No tests required because** there are no existing tests on the
  package, and a `MaxOpenConns(1)` setting is an observable
  property of the returned `*sql.DB` that can be unit-tested
  trivially if the maintainers decide to. Author acknowledges this.

## Risks / nits

1. **Latency floor under burst.** With `MaxOpenConns(1)`, every
   write — `session.Save`, `message.Append`, `filetracker.Update`
   — serializes. For a single user this is invisible (writes are
   << 1ms on a local SQLite WAL). For parallel sub-agents emitting
   tool-results in parallel, the queue can build to the point where
   `busy_timeout=30s` becomes load-bearing. If anything in the
   call graph holds a write transaction while waiting on a network
   call (model API, tool exec), the 30s budget is *one* slow tool
   call away from blowing.
2. **Read parallelism dies too.** `SetMaxOpenConns(1)` doesn't
   distinguish reads from writes. SQLite WAL allows concurrent
   readers + one writer; this PR throws that capability away.
   `SetMaxOpenConns(1)` is *strictly* more conservative than
   needed. A more nuanced fix would be a separate read-only pool
   (`PRAGMA query_only`) and a single-writer pool. Out of scope
   for this PR but worth flagging.
3. **No observability when contention bites.** When the busy
   timeout starts firing, the symptom will be "session save took
   30s" with no specific log signal. Adding a one-line `slog.Warn`
   when `db.Stats().WaitDuration` exceeds a threshold would give
   future debugging a chance.
4. **Competes with #2690.** As noted above, #2690 is the
   belt-and-suspenders version of the same fix. From a project-
   hygiene standpoint, the maintainers need a clear "which PR is
   we landing" decision before merging either. Otherwise they risk
   a later-merging PR that conflicts trivially or that re-adds the
   `_txlock=immediate` change as a follow-up to a fix that should
   have included it from the start.
5. **No regression test that proves the fix.** A reproducer
   (`go test -race` with N goroutines hammering the DB and a
   cancel mid-transaction) would lock in the property that future
   refactors can't silently regress. Hard to write but valuable.

## Verdict — needs-discussion

This is a *good* fix in isolation. The reason for `needs-discussion`
rather than `merge-after-nits` is the duplicate-PR situation with
#2690: maintainers should pick one (probably #2690, since the
`_txlock=immediate` half is a real second mitigation), and close
the other with a comment crediting both authors. If the maintainers
decide to land #2691 *instead* of #2690, the suggestions above
about `_txlock=immediate` and a wait-duration log signal should be
folded in as follow-ups.

## What I learned

Two PRs targeting the same root cause but with different fix
shapes is a recurring OSS situation that often resolves badly: one
PR merges, the other is closed without explanation, and the loser's
extra mitigation (here: `_txlock=immediate`) gets lost. The
healthier resolution is for the reviewer to explicitly pick one and
either (a) port the unique parts of the loser into the winner
before merging, or (b) ask the loser's author to rebase onto the
winner's branch and add their unique mitigation as a follow-up
commit. Either way the discussion belongs in PR comments, not in a
silent merge. Separately, `SetMaxOpenConns(1)` against SQLite is a
common mitigation that's almost always *too coarse* (it kills read
parallelism); the right long-term shape is a read pool + a writer
pool, but that's a much bigger refactor and not justified by the
data on hand.
