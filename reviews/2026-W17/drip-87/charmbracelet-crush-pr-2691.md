# Review — charmbracelet/crush#2691: fix(db): cap SQLite pool to one writer to prevent NOTADB corruption

- **Repo:** charmbracelet/crush
- **PR:** [#2691](https://github.com/charmbracelet/crush/pull/2691)
- **Author:** SAY-5 (Sai Asish Y)
- **Head SHA:** `0b82b179a9c64a7075ef0074f82abc26cdcfd089`
- **Size:** +12 / −0 across 1 file (`internal/db/connect.go`)
- **Verdict:** `needs-discussion`

## Summary

Adds `db.SetMaxOpenConns(1)` to the SQLite connection setup at `internal/db/connect.go:48` along with a 12-line block comment explaining the failure mode: parallel sub-agents (session, message, history, filetracker) sharing the `*sql.DB` previously opened an unbounded number of concurrent SQLite handles via Go's default pool, and interleaved WAL frames + auto-checkpoints from those handles plus mid-checkpoint cancellation produced `SQLITE_NOTADB (26)` errors that corrupted the project session. The fix caps the pool at one writer; the existing `busy_timeout` (30s) queues concurrent callers.

## Technical assessment

The diagnosis in the inline comment at `:48-58` is plausible and the fix is minimal. SQLite-with-WAL is single-writer / multi-reader by design, but `database/sql` defaults to an unbounded pool and Go's WAL-mode SQLite drivers (the `mattn/go-sqlite3` and `modernc.org/sqlite` families) both support multiple writer handles per process — they just serialize through the kernel/sqlite3 itself, with `SQLITE_BUSY` retries governed by `PRAGMA busy_timeout`. The reported corruption (`SQLITE_NOTADB`, error code 26) is an unusual outcome and the linked failure mode — "interleaved WAL frames + auto-checkpoints, combined with mid-checkpoint cancellation (context cancel, SIGINT, OOM), desynced the main DB file from the WAL" — is the kind of thing that *can* happen but usually requires either (a) a buggy driver that doesn't hold the WAL lock during checkpoint, (b) a kernel-level filesystem issue, or (c) actual concurrent process-level access (not just goroutine-level).

That's why this is `needs-discussion` rather than `merge-as-is`:

**(1) Is the diagnosis right?** SQLite with WAL and `journal_mode=WAL` + `synchronous=NORMAL` should not corrupt under multi-handle concurrent access from a single Go process — that's the whole point of WAL. If users are seeing `SQLITE_NOTADB`, the more common root causes are: (i) the DB file was being concurrently accessed by another process (e.g. a previous crush instance that didn't fully release the lock), (ii) the underlying filesystem is APFS-cloned or on a network mount that doesn't support `fcntl` locks correctly, (iii) the `mattn/go-sqlite3` build was compiled without `SQLITE_THREADSAFE=1`. Capping `MaxOpenConns(1)` will hide all of those, but won't fix the underlying invariant violation.

**(2) Performance impact.** With `SetMaxOpenConns(1)`, every SQL operation across every sub-agent — session writes, message writes, history reads, filetracker reads — serializes through a single connection guarded by Go's `database/sql` mutex. A 30s `busy_timeout` is irrelevant at this layer because contention is now in Go, not in SQLite. For a single-user CLI this is probably fine (writes are infrequent and small), but it means parallel reads from sub-agents will block each other, which was a major architectural reason to use SQLite-WAL in the first place.

**(3) Reader/writer asymmetry not exploited.** SQLite's WAL mode allows N concurrent readers + 1 writer. The right fix, if the diagnosis is real, is two `*sql.DB` handles: one for writes (`SetMaxOpenConns(1)`) and one for reads (default pool). Calling code routes through the appropriate one. That preserves read concurrency while serializing writes. Alternatively, a single connection pool with `SetMaxOpenConns(1)` and `SetMaxIdleConns(1)` plus separate prepared-statement readers via `BeginTx(ctx, &sql.TxOptions{ReadOnly: true})` — but that's more invasive.

**(4) Is `busy_timeout` actually doing anything now?** The comment says "busy_timeout (30s) queues concurrent callers while this handle is in use." That's not quite right — with `MaxOpenConns(1)`, callers queue in Go's `database/sql` mutex, not in SQLite's `busy_timeout`. `busy_timeout` only matters when SQLite itself returns `SQLITE_BUSY`, which won't happen with a single connection. The comment should be corrected to avoid future confusion.

## What would change the verdict

- A reproducer (even a flaky one) for `SQLITE_NOTADB` under crush's actual usage pattern. Without that, this is a "felt slow / saw corruption once / capped the pool" change that may be papering over a different bug.
- Confirmation that the SQLite driver in use is built with `SQLITE_THREADSAFE=1`. If it isn't, that's the real fix.
- Benchmark numbers showing the read-throughput regression is acceptable for typical sessions (e.g. 10 sub-agents reading message history simultaneously). Even a microbenchmark would suffice.
- An alternative formulation as a 2-pool reader/writer split (item 3 above), which preserves the win without sacrificing read concurrency.

## Verdict rationale

`needs-discussion`. The fix is one line, the comment is well-written, and the failure mode (NOTADB corruption) is genuinely scary. But the diagnosis ("multi-handle WAL writers desync") is unusual enough that I'd want either a reproducer or a deeper RCA before accepting that the right fix is "serialize all DB access through a single connection." There's a good chance the real bug is elsewhere (driver build flags, concurrent process access, filesystem) and this PR only masks it. Companion PR #2690 (taoeffect, "fix(db): prevent SQLITE_NOTADB corruption under concurrent sub-agents") suggests the same symptom has at least two competing fixes — that's exactly the kind of situation where a maintainer should pick the right one rather than merge whichever lands first.
