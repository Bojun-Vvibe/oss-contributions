# Review: charmbracelet/crush #2691 — fix(db): cap SQLite pool to one writer

- Repo: charmbracelet/crush
- PR: #2691
- Head SHA: `0b82b179a9c64a7075ef0074f82abc26cdcfd089`
- Author: SAY-5
- Size: +12 / -0 across 1 file

## What it does
Sets `db.SetMaxOpenConns(1)` on the SQLite `*sql.DB` returned by `Connect`,
serializing all access through a single connection. Includes a 12-line
comment explaining the SQLITE_NOTADB (26) corruption mode this guards
against (parallel sub-agents, interleaved WAL frames, mid-checkpoint
SIGINT/cancel).

## File-level notes

**`internal/db/connect.go` @ L45+ (head `0b82b17`)**
```go
+ // SQLite is a single-writer database. Parallel sub-agents (...)
+ // ... cap the pool at one writer; busy_timeout (30s) queues concurrent
+ // callers while this handle is in use.
+ db.SetMaxOpenConns(1)
```

- The comment is excellent — explains *why*, not just *what*, and names
  the failure mode (SQLITE_NOTADB 26) so future readers searching for
  the symptom find this fix.
- Correctness: with WAL mode, SQLite supports concurrent readers but only
  one writer. Capping at 1 sacrifices read parallelism for safety. For
  a CLI agent's local DB this is the right trade-off; the
  `busy_timeout=30s` already configured upstream means callers will
  queue rather than fail.
- Potential perf concern: any long-running read (e.g. a streaming
  history scan) will now block all other callers, including writes,
  for its entire duration. If reads start to dominate, a follow-up
  splitting reader/writer connections (`SetMaxOpenConns(N)` on a
  read-only `*sql.DB` opened with `?mode=ro`, separate writer DB at
  `MaxOpenConns(1)`) is the standard pattern. Not blocking for this PR.
- No test added. A reproducer for the corruption mode is hard to write
  reliably; skipping is acceptable. A minimal sanity test asserting
  `db.Stats().MaxOpenConnections == 1` after `Connect` would at least
  pin the contract.

## Risks
- Throughput regression for read-heavy workloads — likely small in
  practice for a single-user CLI but worth a follow-up benchmark.
- No risk of correctness regression; this is strictly safer than before.

## Verdict: `merge-as-is`
Critical correctness fix with a great explanatory comment. The perf
follow-up (split reader/writer pools) can be a separate issue.
