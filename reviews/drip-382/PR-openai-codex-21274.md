# openai/codex PR #21274 — [codex] Deduplicate invalid skill load warnings

- URL: https://github.com/openai/codex/pull/21274
- Head SHA: `d80e27f888fd64760d9c006d986d896e6de3a3c4`
- Size: +80 / -1

## Summary

Suppresses duplicate "⚠ Skipped loading N skill(s)…" history-cell emissions
when the same set of `SkillsListResponse` errors recurs across consecutive
skill refreshes. Tracks the last-emitted `Vec<SkillErrorInfo>` per `cwd` on
the TUI `App`, re-emits only when the batch *changes*, *clears*, or
*recurs after clearing*. Without this, every skill-list refresh (which
fires on init, file change, and a few other paths) was repainting the same
warning batch into the transcript on each call.

## Specific findings

- `codex-rs/tui/src/app.rs:467` — new field
  `skill_load_warnings_by_cwd: HashMap<PathBuf, Vec<SkillErrorInfo>>` on
  `App`. `PathBuf` keying is correct: per-cwd is the granularity that
  matters (a multi-root TUI session shouldn't have `cwd1`'s cleared warnings
  silence `cwd2`'s fresh ones).
- `codex-rs/tui/src/app.rs:890` — initialized to `HashMap::new()` in the
  main constructor; mirrored in `app/test_support.rs:42` and the two
  `make_test_app` helpers in `app/tests.rs:3861, 3925`. All four
  construction sites updated — important because forgetting a single one
  would compile-fail (good) but a future variant would silently regress.
- `codex-rs/tui/src/app/thread_routing.rs:1322-1324` — old
  `emit_skill_load_warnings(&self.app_event_tx, &errors)` call site
  replaced with `self.emit_skill_load_warnings_if_changed(&cwd, errors)`.
  Single call-site swap, low risk.
- `thread_routing.rs:1327-1344` — new `emit_skill_load_warnings_if_changed`
  has three branches: (a) `errors.is_empty()` removes the cwd entry and
  returns silently — correct, this is the "errors cleared" reset; (b)
  exact-equality match against the cached previous batch via
  `is_some_and(|previous_errors| previous_errors == &errors)` — relies on
  `SkillErrorInfo: PartialEq`, which the diff implicitly assumes; (c) any
  other case emits + caches. The `==` semantics include `path`, `message`
  and any other fields, so a SKILL.md path move that happens to produce
  the same `message` text is treated as a new error and re-emitted, which
  is the right call.
- `codex-rs/tui/src/app/tests.rs:2462-2516` — new
  `repeated_skill_load_warnings_emit_once_until_errors_clear` test pins
  exactly the contract: (1) two identical responses → one batch emitted;
  (2) empty response → no emission, cache cleared; (3) response with
  errors again after clear → re-emits. The
  `drain_insert_history_transcripts` helper at `:2519-2528` is a nice
  test-utility extraction.

## Notes

- Cache is keyed by `PathBuf`. If `cwd` is ever observed both as a
  symlink and as its canonical path within one process, the dedupe key
  would split — would cause duplicate warnings in that edge case. Probably
  fine in practice (TUI never observes both forms), but a one-line comment
  noting the assumption would help the next reader.
- `HashMap` grows monotonically per-distinct-cwd over the session
  lifetime. The TUI is short-lived enough that this isn't a leak in
  practice, but a `LruCache` or similar bounded structure would be more
  defensive — not a blocker.
- The PR body notes "2 existing status permission-profile tests failed
  because `/etc/codex/requirements.toml` rejects `DangerFullAccess`" — pre-
  existing environmental failure, unrelated to this diff. Worth a
  maintainer-side confirmation that those failures reproduce on `main`
  before merging.

## Verdict

`merge-after-nits`
