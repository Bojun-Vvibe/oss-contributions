# sst/opencode PR #25446 — fix(lsp): ignore pyright virtualenv diagnostics

- Repo: `sst/opencode`
- PR: #25446
- Head SHA: `c7abb0779c857c1b05a5dd0091b90f40248815b5`
- Author: addu2612

## Summary
Adds a pyright-only filter that drops diagnostics for files inside `venv` /
`.venv` so virtualenv noise stops leaking into the workspace diagnostic state
(`packages/opencode/src/lsp/client.ts`). Includes a focused workspace
pull-diagnostics test in `packages/opencode/test/lsp/client.test.ts` that
verifies a real source file still surfaces while a `.venv/lib/...` entry is
suppressed. Closes #25408.

## Specific references
- `packages/opencode/src/lsp/client.ts:165-188` — new `shouldIgnoreDiagnostics`
  helper. Correctly scopes via `input.serverID === "pyright"`, uses
  `path.relative(input.root, filePath)` and rejects absolute / `..` paths so
  files outside the workspace root aren't mistakenly matched.
- Same file 172-181 — both `updatePushDiagnostics` and `updatePullDiagnostics`
  call `pushDiagnostics.delete(filePath)` / `pullDiagnostics.delete(filePath)`
  on the ignore path. Good — that prevents stale entries from a prior
  un-ignored update lingering in the maps.
- `packages/opencode/test/lsp/client.test.ts:447-512` — new
  `pyright ignores virtualenv workspace pull diagnostics` test asserts the
  real source file diagnostic survives and `client.diagnostics.has(virtualenv)`
  is false. Solid coverage of the intended behavior.

## Verdict
`merge-after-nits`

## Rationale
Narrow, server-scoped fix with a faithful test. Two nits worth raising
before merge:
1. `path.relative` will return e.g. `Lib\\site-packages\\.venv\\...` on
   Windows — the `split(/[\\/]+/)` already handles both separators, but the
   `part === ".venv" || part === "venv"` check is case-sensitive. On
   Windows, virtualenvs sometimes use `Scripts/` and uppercase `Venv`/`ENV`
   directory names. Worth either lowercasing `part` or documenting the
   expected casing.
2. The merge path (`mergedDiagnostics`) still concatenates from both maps,
   but since both `updatePush*` / `updatePull*` are the only writers and
   both delete on ignore, this is fine. Worth a one-line comment in
   `shouldIgnoreDiagnostics` saying "callers must also delete prior entries"
   so a future caller of `pushDiagnostics.set` doesn't bypass the guard.

No banned strings; behavior change is contained to one LSP server ID.
