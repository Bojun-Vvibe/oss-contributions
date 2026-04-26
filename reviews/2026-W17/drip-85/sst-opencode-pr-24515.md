# sst/opencode PR #24515 — feat(tool): add patch_file, ast_query, ast_edit — hash-anchored + AST-native editing

- **Repo:** sst/opencode
- **PR:** [#24515](https://github.com/sst/opencode/pull/24515)
- **Head SHA:** `2fbc712e50fd42d68af4a905467256910c225fad`
- **Author:** r3vs
- **Size:** 9 files, +871/−6
- **Closes:** #24511

## Context

Three new built-in tools intended to reduce token usage and improve edit
precision on large codebases, inspired by the Dirac coding agent. The PR
introduces a TypeScript surface — `patch_file` (hash-anchored parallel
edits), `ast_query` (tree-sitter S-expression search), and `ast_edit`
(byte-offset replacement of a range from an `ast_query` result) — and
wires them into `tool/registry.ts:203-232` alongside the existing
`edit`/`write`/`apply_patch` family.

## What the diff actually does

**New files**
- `packages/opencode/src/ast/languages.ts` (32 lines) — `LANGUAGE_MAP`
  for ts/tsx/js/python/bash WASM grammars resolved via `createRequire`
  + `require.resolve("<pkg>/package.json")` then string-replacing
  `package.json` with the WASM filename.
- `packages/opencode/src/ast/parser.ts` (166 lines) — `AstParser`
  Effect service exposing `parse`/`query`/`queryFile`/`nodeAtRange`
  with a module-level `_grammarCache: Map<string, any>` and a lazy
  `loadTreeSitter()` singleton.
- `packages/opencode/src/tool/patch_file.ts` (266 lines) — anchor
  computation via `crypto.createHash("sha256").update(window).digest("hex")`
  on a `2*contextLines+1` line window, anchor map builder that emits
  one anchor per file line, `resolvePatch` linear scan of every line,
  reverse-sorted application via `applyPatches`.
- `packages/opencode/src/tool/ast_edit.ts` (144 lines) — byte-offset
  replacement using `TextEncoder`/`TextDecoder` slice/concat at
  `nodeInfo.start_byte`/`end_byte` plus optional
  `verify_node_type` guard.
- `packages/opencode/src/tool/ast_query.ts` (78 lines).
- Three corresponding `*.txt` description files.

**Modified**
- `packages/opencode/src/tool/registry.ts` — registers the three new
  tools at `:203-232`, adds them to the always-on availability filter
  at `:294-298`, and adds `AstParser.Service` to the layer requirement
  list at `:93` plus `AstParser.defaultLayer` to `defaultLayer` at
  `:357`. Also drops three unrelated comment blocks at `:122-130`,
  `:171-172` (cosmetic noise that should not be in this PR).

## Strengths

- The `patch_file` safety design at `patch_file.ts:163-200` is solid:
  pre-resolution of all anchors before any write, hard error on
  anchor miss with a clear "re-run with `compute_anchors=true`"
  remediation, overlap detection on `sortedResolved` at `:182-192`,
  reverse-order application via `[...resolved].sort((a,b) => b.startLine - a.startLine)`
  at `:144` to prevent offset drift.
- The full edit pipeline is preserved end-to-end —
  `ctx.ask({ permission: "edit", … })` → `afs.writeWithDirs` →
  `format.file` → `bus.publish(File.Event.Edited)` →
  `lsp.touchFile` → diagnostics collection — at `patch_file.ts:213-244`
  and `ast_edit.ts:115-138`. Same guarantees as `edit`/`write`,
  no parallel write path.
- BOM preservation is correct: `Bom.readFile` → `Bom.split(contentNew)` →
  `desiredBom = source.bom || next.bom` → `Bom.join` and
  `Bom.syncFile` after format, mirroring `edit.ts`.
- `ast_edit` doing byte-offset slicing via `Uint8Array.set` in
  three contiguous regions (`ast_edit.ts:97-105`) is the right
  primitive — won't drift on whitespace because the AST already
  resolved the byte range.

## Risks / nits

1. **`buildAnchorMap` is O(N) hashes per file** — every line gets
   a `2*contextLines+1` window hashed (default 11 lines per anchor).
   For a 5000-line file that's 5000 SHA-256 invocations per
   `compute_anchors` call. Consider sampling (every 5 lines) or
   exposing a `line_range` filter so the model can scope.
2. **`resolvePatch` is also O(N) per patch** (`patch_file.ts:121-141`).
   With K patches that's O(N·K). For multi-edit operations on a large
   file this dominates the cost. Build a `Map<hash, lineNumber>` once
   from `buildAnchorMap`, reuse for all patches.
3. **Anchor map preview is single-line** (`patch_file.ts:84`,
   `preview: lines[i].slice(0, 80)`) — the model has to pick anchors
   by 80-char line previews, which is fragile when many similar
   lines exist (e.g., closing braces). Include a 3-line context
   preview or hash-of-window.
4. **`compute_anchors` and `patches` modes share a single tool entry**.
   The `compute_anchors` branch returns at `patch_file.ts:215` with
   `metadata: { anchors }` and `output: JSON.stringify(anchors, null, 2)`
   — that JSON could be megabytes for a large file, blowing past
   the model's context budget the very thing this tool is meant to
   conserve. Either truncate or split into two tools.
5. **`ast_query` capture-name extraction uses children iteration**
   at `parser.ts:117-124`, looking for the first `identifier` /
   `type_identifier` / `property_identifier` child. For
   `(method_definition) @method` the name lives on `node.childForFieldName("name")`
   which is the documented tree-sitter API; the children-walk works
   for most cases but will return the wrong identifier on
   destructured method shorthand or private (`#name`) fields. Use
   the field-accessor API.
6. **`languages.ts:18-24` does string substitution on a resolved path**
   (`require.resolve("<pkg>/package.json").replace("package.json", wasmFile)`).
   That breaks if the package's `package.json` lives in a path that
   contains the literal `package.json` substring elsewhere — unlikely
   but not impossible — and silently falls back to the bare WASM
   filename on `require.resolve` failure (`return wasmFile`), which
   then fails opaquely at runtime when `Language.load` can't find
   it. Throw on resolve failure.
7. **`nodeAtRange` walks the entire tree on every call** with no
   pruning by `startPosition.row` (`parser.ts:131-142`). Skip
   subtrees whose range doesn't contain the target.
8. **Three new TS files share one `_Parser`/`_grammarCache` module
   global** (`parser.ts:13-15`). The Effect `Layer` model expects
   service state to be encapsulated; module globals undermine
   testability and the singleton is keyed by process not by
   workspace, which is fine here but worth a comment.
9. **`registry.ts:122-130` deletes three comment blocks**
   (the `Plugin tools define their args as a raw Zod shape…`
   comment, the `match is an absolute filesystem path…` comment,
   and the function-doc on `fromPlugin`) that have nothing to do
   with the three new tools. Either keep them or split into a
   `chore: drop stale comments` PR.
10. **No test coverage at all** in the diff. For a 871-line tool
    addition that does write-with-permission this is the biggest
    gap. At minimum: anchor-mismatch unit test, overlap-detection
    unit test, byte-offset round-trip test for `ast_edit`,
    multi-language smoke test for `ast_query`.
11. **Tools always available** (`registry.ts:294-298`) regardless of
    model — the existing `apply_patch`/`edit` split is gated by
    `usePatch = input.modelID.includes("gpt-")…`. With three new
    tools always exposed, every model now sees `edit`, `apply_patch`,
    `patch_file`, `ast_query`, `ast_edit` as overlapping options.
    The model's "which tool should I use" decision tree gets noisier.
    Either gate behind a flag or document the picking rules in the
    description.

## Verdict

`request-changes`

Concept is good and the safety properties of `patch_file` are well
thought through, but: zero tests on a 871-line tool addition that
performs writes, the `compute_anchors` JSON output potentially
defeats the very token-saving purpose of the tool, three unrelated
comment-block deletions in `registry.ts` have no place in this PR,
and the always-on registration produces tool-overlap noise across
five edit-class tools. Split out the comment deletions, add at
minimum 4-5 unit tests covering the safety invariants, return
truncated/sampled anchors instead of the full O(N) map, and decide
whether `patch_file`/`ast_*` should be flag-gated or replace the
existing `edit` entirely.
