# google-gemini/gemini-cli PR #26535 — Tighten private Auto Memory patch allowlist

- URL: https://github.com/google-gemini/gemini-cli/pull/26535
- Head SHA: `34ea5b5f7dacccefae5d6bbdf85f0d341d87831b`
- Size: +288 / -26

## Summary

Closes #26520 — narrows the private-tier Auto Memory patch validator so
patches can only target the project memory document set: `MEMORY.md` plus
direct sibling `*.md` files in the project memory directory. Previously the
"directory root" allowlist was coarse: anything resolving inside the root
was allowed, including `.extraction-state.json`, `.extraction.lock`,
`.inbox/...`, `skills/...`, non-markdown files, and nested-subdir markdown.
Same hardening applied to the global tier so child paths underneath the
`~/.gemini/GEMINI.md` single-file allowlist (i.e. `GEMINI.md/nested.md`)
are also rejected. The new logic shares one kind-aware target validator
across patch *listing*, *aggregate apply/dismiss*, and *direct apply*.

## Specific findings

- `packages/core/src/commands/memory.ts:484-505` — new
  `MemoryPatchTargetValidationContext` carries `kind`, the existing
  `allowedRoots`, plus kind-specific `privateMemoryDirs` and
  `globalMemoryFiles`. `getMemoryPatchTargetValidationContext()` at
  `:524-558` resolves both the raw and canonicalized variants and dedupes
  via `uniqueResolvedPaths`. The dual-resolution dance is the right
  defensive shape because contributors with `~/Library/...` symlinks
  (macOS) or `/private/...` (also macOS) would otherwise bypass the
  containment check.
- `memory.ts:507-522` — `isAllowedPrivateMemoryFileName` requires the
  filename to either be the literal `PROJECT_MEMORY_INDEX_FILENAME`
  (`MEMORY.md`) or a non-dotfile with a `.md` suffix. Correctly rejects
  `.extraction-state.json` (extension wrong + dotfile), `.extraction.lock`
  (dotfile), and `.inbox/review.md` (the `.inbox/` directory is not the
  immediate parent of the basename, so `path.dirname(resolvedTargetPath)
  !== memoryDir` filters it out at `isAllowedPrivateMemoryDocumentPath`
  `:530-538`). `skills/SKILL.md` similarly fails the parent-dir equality.
- `memory.ts:572-606` — `resolveMemoryPatchTargetWithinAllowedSet` now
  applies the kind-specific predicate to *both* the input `targetPath`
  *and* the canonical `resolvedTargetPath`. This double-check is what
  blocks the symlink-bypass case (`./MEMORY.md → ./.extraction.lock`):
  the input passes the canonical-set check but the resolved path doesn't.
  Correct construction.
- `memory.ts:609-628` — `findDisallowedMemoryPatchTarget` walks
  `validateParsedSkillPatchHeaders(parsedPatches)` and returns the first
  bad target path so callers can surface it in error messages. Honest
  early-exit; doesn't try to enumerate all violations in one pass which
  is fine for a UX-facing rejection.
- `packages/core/src/commands/memory.test.ts:469-510` — new
  `rejects private patches that target in-root non-memory documents`
  enumerates the six adversarial targets (`.extraction-state.json`,
  `.extraction.lock`, `.inbox/private/review.md`, `skills/generated/SKILL.md`,
  `notes.txt`, `nested/topic.md`) and asserts both
  `listInboxMemoryPatches` returns zero patches *and*
  `applyInboxMemoryPatch` rejects each with the exact
  `outside the private memory root or target allowlist` error message.
  After-rejection `fs.access(...).rejects.toThrow()` confirms no file was
  written to disk — which is the actual security property. Strong test.
- `memory.test.ts:533-541` and `:921-955` — symmetric global-tier
  `nested.patch` cases pin the "child of the single-file allowlist" reject
  (the `~/.gemini/GEMINI.md/nested.md` shape).
- `memoryPatchUtils.ts:18 / +14` change — `applyParsedPatchesWithAllowedRoots`
  signature accepts an optional resolved-target predicate. Lets memory
  tier narrow the existing skill-tier root allowlist without changing
  skill-patch behavior. Backward-compatible.

## Notes

- Validator still allows any non-dotfile `*.md` directly inside the
  project memory dir. That's the intended product behavior per #26520, but
  worth a one-line maintainer confirm that names like `.MEMORY.md`,
  `MEMORY.md.bak`, `MEMORY.MD` (uppercase) are intentionally
  outside/inside the allowlist (current logic: dotfile rejected, `.bak`
  rejected by extension check, uppercase `.MD` *accepted* via the
  `.toLowerCase().endsWith('.md')` branch — probably correct but should be
  noted).
- `uniqueResolvedPaths` uses `Set` over `path.resolve(...)` strings; on
  case-insensitive filesystems (default macOS HFS+/APFS, default Windows
  NTFS) this won't dedupe `MEMORY.md` and `memory.md` even though they
  refer to the same file. Probably moot here since these paths come from
  config + system APIs, not user input.
- No coverage of the resolved-target-predicate API surface from the
  *skill* tier's perspective (i.e. confirming skill patches still work
  end-to-end with the new optional parameter). The PR claims
  "skill_extraction.eval.ts: 4 passed, 1 skipped" which exercises this
  indirectly — fine.
- New helper functions (`isAllowedPrivateMemoryDocumentPath`,
  `isAllowedGlobalMemoryDocumentPath`, etc.) are not exported. Good — keeps
  the public memory-command surface from accruing new symbols. If unit
  tests for those helpers in isolation are needed later, they can be
  exported with `__internal__` prefix.

## Verdict

`merge-after-nits`
