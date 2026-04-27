# charmbracelet/crush#2691 — fix(db): cap SQLite pool to one writer to prevent NOTADB corruption

- **Head**: `0b82b179a9c64a7075ef0074f82abc26cdcfd089`
- **Size**: +12/-0 in `internal/db/connect.go`
- **Verdict**: `merge-as-is`

## Context

Refs upstream issue #2682. The shared `*sql.DB` returned by `db.Connect` is used by every service (session/message/history/filetracker) and every sub-agent. With Go's default unbounded `MaxOpenConns`, parallel sub-agents can hold multiple concurrent SQLite handles. Mid-WAL-checkpoint cancellation (context cancel, SIGINT, OOM) on any one of those handles can desync the main DB file from the WAL, surfacing on next open as `SQLITE_NOTADB (26)` — bricking the project session until `.crush/crush.db` is manually deleted.

## Design analysis

The fix at `internal/db/connect.go:48-57` is a single SQL pool tweak right before the `PingContext` smoke:

```go
// SQLite is a single-writer database. Parallel sub-agents (session,
// message, history, and filetracker services share this *sql.DB)
// previously opened an unbounded number of concurrent SQLite handles
// through Go's default pool. Interleaved WAL frames + auto-checkpoints
// from those handles, combined with mid-checkpoint cancellation
// (context cancel, SIGINT, OOM), desynced the main DB file from the
// WAL and surfaced at next open as SQLITE_NOTADB (26) — making the
// project session unrecoverable without deleting .crush/crush.db.
// Cap the pool at one writer; busy_timeout (30s) queues concurrent
// callers while this handle is in use.
db.SetMaxOpenConns(1)
```

This is the textbook fix for shared-`*sql.DB`-against-SQLite. Three things to confirm:

1. **`busy_timeout` is the queueing mechanism.** PR body and comment both reference it as the existing 30s setting; that means a caller that arrives while the single connection is in use blocks (up to 30s) inside the driver before returning a `SQLITE_BUSY`. With `MaxOpenConns=1`, the queueing actually happens at Go's `database/sql` level (`db.Conn` waits for the in-use handle to be released) rather than via SQLite's busy retry, but the comment is still substantively correct: nothing at call sites needs to change.
2. **No write-vs-read split.** WAL mode normally lets readers proceed concurrently with a writer, and `MaxOpenConns=1` gives that up. The trade-off is justified — the corruption surface that motivated the PR is the failure mode under cancellation, and the relief from "safely accept N concurrent readers" doesn't help on an agent's home directory database where the working set is small and hot in OS page cache.
3. **`busy_timeout=30s` ceiling.** A pathological writer (e.g. a long sub-agent session writing thousands of messages in a single transaction) could now starve other services for up to 30s. PR body acknowledges this implicitly by leaving call sites unchanged. Worth a follow-up: consider splitting filetracker (which is mostly reads + occasional batched writes) onto its own `*sql.DB` if real workloads start hitting the 30s wall.

## Verdict reasoning

This is a clean, surgical fix to a real corruption mode with a documented user-recovery cost (manual `.crush/crush.db` delete). The comment block at the call site is explicit about cause-and-effect, the PR description traces the chain end-to-end, and the change is one method call. The only honest concern (potential 30s starvation of secondary services) is a follow-up problem, not a fix-blocker.

## Tests

PR body: `go build ./internal/db/...` clean; package has no existing tests. Deterministically reproducing the SQLITE_NOTADB corruption requires injecting cancellation between WAL frame write and checkpoint completion, which isn't trivial and probably isn't worth a custom test harness in this package — the fix mechanism is well-known SQLite-+-Go folklore. Acceptable.

## What I learned

`db.SetMaxOpenConns(1)` is one of those single-line fixes that's almost always more correct than whatever clever alternative is on the table for embedded SQLite under concurrent use. The interleaved-WAL-frames-into-a-mid-cancellation-checkpoint failure mode is exactly the kind of bug that takes hours to root-cause and one line to fix once you know the shape. The PR description's CAUSE → CONSEQUENCE → FIX → COMMENT structure is also a nice template — the inline comment at `:48-57` will save the next maintainer from "why is this hardcoded to 1?" archaeology.
