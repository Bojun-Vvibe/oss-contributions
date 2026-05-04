# openai/codex PR #21069

- **Title**: Spill large hook outputs from context
- **Author**: abhinav-oai
- **Head SHA**: `468fcead29898f1b4e36ca926126849358744699`
- **Diff**: +424 / -2

## Summary

Adds a hook-output budget (2,500 tokens) and spills oversized hook text to `$CODEX_HOME/hook_outputs/<thread_id>/<uuid>.txt`, replacing the model-visible payload with a head/tail truncated preview plus a recovery path. Includes a 3-day retention cleanup keyed off rollout-file mtime.

## File-by-file

### `codex-rs/core/src/hook_output.rs` (new, L1-173)

- **L23-25**: constants are well-named and have reasonable defaults (2,500 tokens / 3-day retention). Worth surfacing as config later but fine as constants for v1.
- **L32-66 `cap_model_visible_hook_text`**: clean control flow — early return on small output, then `create_dir_all` → `write` → fire-and-forget cleanup. Both fs failure paths fall back to in-memory `formatted_truncate_text`, so a broken disk degrades gracefully instead of blocking the model. Good.
- **L79-85 `spilled_hook_output_preview`**: critical detail done correctly — the footer is budgeted *before* truncation via `saturating_sub(approx_token_count(&footer))`. This prevents the recovery-path footer from itself overflowing the budget. This is the kind of bug that usually ships.
- **L92-151 `cleanup_stale_hook_outputs`**: skips active thread (L116-118), removes orphaned dirs lacking a rollout (L122-125), and deletes only after `HOOK_OUTPUT_RETENTION` has elapsed since rollout mtime (L141-147). Sensible.
- **Concern L107-110**: `if !entry.file_type().await?.is_dir() { remove_hook_output_path(...) }` will silently delete *any* stray file at the top of `hook_outputs/`. If a user (or future code path) drops a `.gitkeep` or README there, it's gone. Consider gating on a known suffix.
- **L141-145**: `now.duration_since(modified).ok().is_some_and(...)` silently skips cleanup when modified > now (clock skew). Fine for cleanup-on-best-effort, but worth a debug log.

### `codex-rs/core/src/hook_output_tests.rs` (new, L1-81 visible)

`write_rollout_stub` fixture (L186-196) and `small_hook_output_remains_inline` test confirm the early-return path. Would want to also see explicit tests for: (a) preview footer fits within budget, (b) cleanup respects active thread, (c) orphan dir removal. Diff was truncated at line 400 so these may exist below.

## Risks

- Spilled files contain raw hook output, which can include secrets that hooks legitimately surface. 3-day retention on disk is a meaningful exposure window; document this in the hook author guide and consider mode-0600 on the spilled file.
- New fs activity per oversized hook + a spawned cleanup task per call — cleanup is unbounded concurrency. Consider a once-per-session guard.

## Verdict

**merge-after-nits**
