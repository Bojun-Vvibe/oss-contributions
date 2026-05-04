# sst/opencode #25662 — fix: match non-ASCII folder names in Open Project search

- SHA: `cf6cbc7a93f8018feb7f3e0a765c8eeae98d80a8`
- State: OPEN, +53/-13 across 4 files
- Files: `packages/app/src/components/dialog-select-directory.tsx`, `packages/opencode/src/file/index.ts`, `packages/ui/src/components/list.tsx`, `packages/ui/src/hooks/use-filtered-list.tsx`
- Closes: #25661

## Summary

Fixes Korean (and other non-ASCII) folder search in Open Project. Two concrete bugs: (a) macOS legacy paths are NFD on disk while browser IME emits NFC, so `fuzzysort`'s code-point compare never matched; (b) IME composition fired `applyFilter` per jamo, wasting SDK calls and flickering the list before the syllable settled. Fix is comprehensive — normalize query and comparison keys to NFC at every entry point, and gate `applyFilter` + Arrow / Ctrl+N/P keys on `isComposing`.

## Notes

- `packages/opencode/src/file/index.ts:625` — query gets `.normalize("NFC")` on input. Good.
- `packages/opencode/src/file/index.ts:640-646` — wrapping items as `{ target, search: target.normalize("NFC") }` and indexing `key: "search"` preserves the original path returned to callers. Correct: filesystems that are NFD-sensitive still resolve via the untouched `target`.
- `packages/app/src/components/dialog-select-directory.tsx:118-124` — `search` field for each row gets NFC; original `absolute` path is untouched. Same pattern as server side, consistent.
- `packages/app/src/components/dialog-select-directory.tsx:175` — caching dirs with a precomputed `search` field is right; avoids per-keystroke normalization cost for large directories.
- `packages/ui/src/components/list.tsx:100` — `isImeComposing` checks `event.isComposing || store.composing || event.keyCode === 229`. The `keyCode === 229` fallback is a known IE/Safari quirk for IME — appropriate cross-browser belt-and-suspenders.
- `packages/ui/src/components/list.tsx:103-107` — `applyFilter` early-returns during composition unless `options?.ref` is set. The `ref` escape hatch is for programmatic resets (refs path) — fine, but worth a comment explaining why programmatic callers bypass.
- `packages/ui/src/components/list.tsx:206-213` — `handleCompositionEnd` uses `requestAnimationFrame` then re-checks `store.composing`. Subtle: this guards against a race where another composition starts in the same frame. Good defensive pattern.
- `packages/ui/src/hooks/use-filtered-list.tsx:39-46` — same target/search wrapper pattern as the server. Consistent.
- Manual repro evidence (`mkdir ~/한국어테스트` + before/after gifs) and `bun turbo typecheck` are documented in the PR body.
- Nit: the substring/normalize work is duplicated across four files. Could factor a `normalizeForSearch` helper, but the duplication is small and crosses package boundaries (app/ui/opencode), so inlined is acceptable.

## Verdict

`merge-as-is` — well-scoped, correct fix with clear repro and good test guidance. The `ref` bypass comment is the only nit, and not a blocker.
