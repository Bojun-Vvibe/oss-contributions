# QwenLM/qwen-code PR #3615 — fix(lsp): doc/isPathSafe + symbolName resolution path

- Repo: QwenLM/qwen-code
- PR: #3615
- Head SHA: `63bdf49b`
- Author: @yiliang114
- Diff: +678/-68 across 4 files — `docs/users/features/lsp.md`, `packages/core/src/lsp/LspServerManager.ts`, `packages/core/src/tools/lsp.ts`, `packages/core/src/tools/lsp.test.ts`

## What changed

Three loosely related LSP improvements bundled into one PR:

1. **`isPathSafe` carve-out in `LspServerManager.ts:636-661`**: previously `isPathSafe(command, basePath)` rejected any command whose `path.resolve(basePath, command)` escaped the workspace. That broke bare names like `clangd` (resolved via `PATH`) and absolute paths like `/opt/llvm/bin/clangd`. New logic: bare command name (no `path.sep`) → allow; `path.isAbsolute` → allow; only relative paths get the in-workspace containment check.
2. **`symbolName` resolution path in `lsp.ts`** for `goToDefinition`/`findReferences`/`hover`/`goToImplementation`/`prepareCallHierarchy`. Tools now accept either `(filePath, line, character)` or `symbolName`, with workspace-symbol search resolving the latter to a position before delegating to the underlying LSP request. New tests in `lsp.test.ts:367-395` cover Windows-separator normalization and the new `(or provide symbolName)` error variants at `:120-141` and `:277-294`.
3. **Docs** in `lsp.md`: removes a broken cross-link to `code.claude.com/docs/en/plugins-reference#lsp-servers`, adds documentation for the `symbolName` parameter, and updates the `command` field description to make the bare-name vs absolute-path policy explicit.

## Specific observations

- The `isPathSafe` carve-out is correct in spirit but worth scrutinizing. The original guard's intent was almost certainly "don't let a tampered `.lsp.json` point at `../../bin/evil` that smuggles outside the workspace." Bare-name allow is fine because `PATH` resolution is the user's own shell environment and trust check elsewhere already gates whether LSP servers start (`LspServerManager.ts:626-635` — the comment notes "Trust checks (workspace trust + user consent) already gate server startup"). Absolute-path allow is also fine for the same reason. The remaining relative-path branch correctly keeps the `path.resolve(basePath, command).startsWith(basePath)` containment check. So the security envelope shrinks only along axes already covered by the trust gate — acceptable.
- The `symbolName` path is a real productivity improvement (the model rarely knows exact `(line, character)` ahead of time and otherwise has to first call `documentSymbol`/`workspaceSymbol`, then a second LSP call). But the resolution semantics deserve a docs note: if `symbolName` matches multiple workspace symbols, what does the tool do — first match, ambiguity error, or all-of-them? `lsp.md` should spell that out so the model knows when to disambiguate vs let the tool pick.
- The error message refactor `"filePath is required for ${operation} (or provide symbolName)."` (lsp.test.ts:120-141) is good — it tells the model the alternative path exists. Also worth landing the analogous "neither filePath nor symbolName provided" terminal error that's strictly better than two separate "X is required" messages.
- Diff scoping: three orthogonal changes in one PR (security policy, tool surface, docs) — each is small enough that the bundling is tolerable, but a maintainer reviewer probably wants to evaluate the `isPathSafe` security change independently from the `symbolName` UX feature. Worth splitting if the contributor is willing.

## Verdict

`needs-discussion`

## Rationale

Two of the three changes are clearly correct (`symbolName` UX, docs cleanup). The `isPathSafe` carve-out is also defensible given the existing trust-gate framing, but it's a security-policy decision that benefits from explicit maintainer signoff rather than rubber-stamping inside a feature PR. Recommend splitting into `fix(lsp): allow bare-name and absolute LSP commands` (security) + `feat(lsp): symbolName resolution for location ops` (feature) + `docs(lsp): tighten parameter docs` (docs), which lets the security change get a focused review.
