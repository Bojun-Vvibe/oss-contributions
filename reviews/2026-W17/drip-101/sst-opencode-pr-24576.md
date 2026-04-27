# sst/opencode PR #24576 — fix: pass workspace symbol query to LSP

- URL: https://github.com/sst/opencode/pull/24576
- Head SHA: `f28e13102a5e616508e8d519f7f963a084947a71`
- Diff: +11 / -3 across 2 files
- Author: external contributor (Hona)

## Context / Problem

The experimental LSP tool always called `lsp.workspaceSymbol("")` regardless of
what the model passed in (`packages/opencode/src/tool/lsp.ts:92` pre-change),
which forced clangd / pyright / gopls to return their full unfiltered symbol
table. For LSP 3.17 `workspace/symbol` the empty string is *valid* but means
"return everything", which is the worst possible default for an LLM-driven
symbol search — it's slow and the result usually overflows the tool-output cap
before the model sees anything useful.

## Design

- Adds `query: Schema.optional(Schema.String)` to `Parameters` at
  `packages/opencode/src/tool/lsp.ts:32-34`. Empty string is still legal and
  intentionally documented as "requests all symbols", matching the LSP spec
  wording for `WorkspaceSymbolParams.query`.
- Threads it through at `lsp.ts:95`: `lsp.workspaceSymbol(args.query ?? "")`.
- Tightens the `execute` signature from a hand-rolled object type to
  `Schema.Schema.Type<typeof Parameters>` at `lsp.ts:46`, so the tool body
  stops drifting from the `Parameters` schema (good cleanup).
- Tool-description text in `lsp.txt` is updated to add a `workspaceSymbol also
  accepts: query` block plus an explicit "filePath is not sent in the LSP
  workspace/symbol request — used to select and start the matching server"
  note. That second sentence is the right thing to call out, because a naive
  reader of the tool schema would assume `filePath` scopes the search.

## Risks / nits

- `Schema.optional(Schema.String)` accepts the empty string and `undefined`
  identically (both fall through to `""`), so the `?? ""` is correct but the
  description "Empty string requests all symbols" is the only place the
  no-filter path is documented. Worth a one-line example in `lsp.txt` showing
  a real partial match, e.g. `query: "user"`.
- No test added, but this tool surface has zero existing tests in the package
  so a single-PR "add tests for this and only this" ask would be unfair.
- Author admits `bun typecheck` (`tsgo --noEmit`) timed out at 10s in their
  worktree and they relied on the push hook's `bun turbo typecheck`. Maintainer
  should still confirm CI green before merging.

## Verdict

`merge-as-is` — surgical 8-line behavior fix that brings the tool in line with
the LSP 3.17 spec, keeps the existing `filePath`-based server-startup contract,
and tightens the param-type binding as a drive-by.

## What I learned

When wrapping an LSP method as an LLM tool, the spec's "empty string is valid"
escape hatch can quietly become the only behavior if the wrapping tool never
exposes the param. Always carry the spec's filter args through to the schema,
even if you keep them optional.
