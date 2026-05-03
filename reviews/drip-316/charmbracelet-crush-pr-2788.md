# Review: charmbracelet/crush #2788 — config: lenient shell expansion default, uniform coverage across MCP, LSP, and providers

- PR: https://github.com/charmbracelet/crush/pull/2788
- Head SHA: `5d1dfa346a774f5c0871d6663cc41eb75a25153d`
- Author: meowgorithm (Christian Rocha)
- Size: +970 / -226

## Summary
Reverses an earlier strict-by-default behavior where unset `$VAR`
references in `crush.json` raised hard errors at config load.
Re-establishes lenient nounset (matching bash) so a bare `$UNSET`
expands to the empty string, then layers on:
- `${VAR:?message}` for callers that genuinely need a hard error,
- empty-header dropping in both MCP `headers` and provider
  `extra_headers`,
- a `extra_body` non-expanding JSON passthrough,
- a security note about `$(...)` running at load time.
The README rewrites the section that previously had a typo
("works in `command`...") into clean prose and adds the security
warning.

## Specific citations
- `README.md:299-323` — replaces the broken paragraph that read
  "Shell-style value expansion (`$VAR`...) and nesting (works in
  `command`, `args`, ..., so file-based secrets like work out of
  the box, so you can use values like..." (note the dangling
  syntax + duplicated "so") with a clean rewrite. The new
  `${VAR:?message}` example using `CODEBERG_TOKEN` is a sensible
  pattern. Worth confirming the implementation actually supports
  `:?` syntax (the bash form) — the diff in `init_test.go` doesn't
  test that branch directly.
- `README.md:319-323` — security note explicitly calls out
  `$(...)` runs at load time before the UI appears. Good and
  necessary; should arguably be more prominent (top of the section,
  not buried).
- `internal/agent/tools/mcp/init_test.go:94-130` — the original
  "http unset var surfaces error" test is renamed and rewritten.
  The new failure-mode coverage uses `$(false)` (a failing
  command-substitution) since unset vars no longer error. The
  parallel "http unset var expands empty" pinning test asserts
  `https://$MCP_MISSING_HOST/api` resolves to `https:///api`
  (which is structurally weird but non-empty, so passes the
  existing non-empty guard). This is intentional but fragile — a
  user who typo's `$MCP_HOST` will get a syntactically-malformed
  URL with no warning.
- `internal/agent/tools/mcp/init_test.go:204-247` — same pattern
  for stdio env: failing-`$(cmd)` errors out, bare unset
  silently expands. The pinning subtest at :226-247 explicitly
  checks `ct.Command.Env` contains `FORGEJO_ACCESS_TOKEN=` (with
  empty value). Comment explicitly cites "design decision #18" —
  good traceability, but the design doc isn't linked.
- `internal/agent/tools/mcp/init_test.go:359-396` — header
  resolution: `$(false)` errors, bare unset is silently
  **dropped** from the round-tripper headers. Different policy
  from env (kept-empty) vs. headers (dropped). The README at line
  308-313 documents this asymmetry. Defensible.

## Verdict
**merge-after-nits**

## Nits
1. The `https:///api` outcome for an unset URL var is silent and
   easy to miss in a config typo. Consider a `tracing::warn!` (or
   Go equivalent) when an MCP `URL` resolves to a syntactically
   invalid form, even though it's non-empty. Empty headers are
   dropped silently — fine; malformed URLs deserve at least a log
   line.
2. The README security note at `:319-323` should move to the top
   of the "Shell-style value expansion" section, not the bottom.
   Users who don't read to the end won't learn that `$(...)` runs
   at load time.
3. Link "design decision #18" referenced in `init_test.go:230` to
   the actual design doc/issue so future readers don't have to
   `git log -S`.
