# Review: charmbracelet/crush PR #2690

- **Title:** fix(db): prevent SQLITE_NOTADB corruption under concurrent sub-agents
- **Author:** taoeffect (Greg Slepak)
- **Head SHA:** `7ff330a285702e3e2fbefe33b62325e899018082`
- **Verdict:** needs-discussion

## Summary

The PR addresses a real corruption scenario — `SQLITE_NOTADB (26)` on
re-open after concurrent sub-agent activity — by making three changes:

1. `db.SetMaxOpenConns(1)` on the shared `*sql.DB` to serialize all
   access through a single connection.
2. `_txlock=immediate` on the modernc driver so writers acquire the
   reserved lock up front.
3. `_txlock=immediate` on the ncruces driver via DSN string (also
   switching `driver.Open` from a bare path to `file:` URI form).

The diagnosis (header desync from interleaved checkpoints) is plausible,
but the chosen fix is heavyweight and worth discussing before merge.

## Specific-line comments

- `internal/db/connect.go:50` — `db.SetMaxOpenConns(1)` is a real
  capability regression, not a nit. Every concurrent reader now blocks
  behind every writer, including long-running queries. SQLite + WAL
  was specifically designed to allow concurrent readers; capping the
  pool to 1 throws that away. The comment justifies it as defensive,
  but the more typical fix for "interleaved writers corrupt the file"
  is to (a) set `SetMaxOpenConns(1)` only for *write* operations via a
  separate `*sql.DB` for writes, and keep readers unconstrained, or
  (b) ensure WAL mode + `synchronous=NORMAL` + `busy_timeout` are
  actually being applied (verify the PRAGMAs land — modernc applies
  via DSN, but drivers can silently ignore unknown ones).
- `internal/db/connect_modernc.go:20-22` — `_txlock=immediate` is a
  good addition and addresses the specific deadlock pattern (a
  deferred read transaction trying to upgrade to write while another
  connection holds the reserved lock). This change alone, plus
  verifying WAL is enabled, may resolve the original symptom without
  needing the pool cap.
- `internal/db/connect_ncruces.go:14-19` — switching to `file:` URI
  form is required for the ncruces driver to parse query parameters.
  Confirm this does not break paths containing `?`, `#`, or `%`
  characters (rare but possible in user-chosen `dataDir`); a quick
  `url.PathEscape` on `dbPath` would harden it.

## Risks / nits

- No test reproduces the original corruption or proves the fix.
  Without that, this is a theory-driven change to hot-path config.
- No measurement of the throughput cost of `MaxOpenConns(1)` under the
  expected sub-agent fan-out.
- Mixing two fixes (pool cap + txlock) makes it hard to bisect which
  one actually resolved the bug if a regression appears later.

## Verdict justification

The `_txlock=immediate` change is clearly correct and well-motivated.
`SetMaxOpenConns(1)` is a much bigger semantic change that may not be
necessary if the txlock fix alone resolves the deadlock pattern.
Recommend splitting into two PRs (or at minimum justifying the pool cap
with a benchmark / repro). **needs-discussion.**
