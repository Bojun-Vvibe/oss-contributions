# openai/codex#21311 — fix: preserve reopened descendants under read denies

- Head SHA: `e01fd530048a089a3eb764d285abfdbde3969a72`
- Author: @viyatb-oai
- Link: https://github.com/openai/codex/pull/21311

## Notes
- `codex-rs/protocol/src/permissions.rs:223` swaps `denied_candidates: Vec<Vec<PathBuf>>` for `exact_candidates: Vec<ResolvedFileSystemEntry>`, which is the right shape to retain per-entry access info instead of collapsing to a binary deny.
- Construction at `permissions.rs:239–256` flattens `normalized_and_canonical_candidates(...)` into `ResolvedFileSystemEntry { path, access }` via `AbsolutePathBuf::from_absolute_path(...).ok()` — silently dropping entries that fail conversion is acceptable here since they couldn't have matched anyway, but worth a `debug!` trace.
- Match logic at `permissions.rs:288–300` correctly switches to `max_by_key(resolved_entry_precedence)` so a more-specific reopen entry wins over a broader deny — this is the actual bug fix.

## Verdict
`merge-after-nits`

Logic is sound and mirrors the main resolver's precedence rules. Add a test that asserts a `reopen` descendant under a denied root is now readable, and consider logging the dropped-conversion case in `flat_map`.
