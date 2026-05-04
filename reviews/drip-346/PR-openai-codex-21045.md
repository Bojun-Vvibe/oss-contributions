# openai/codex#21045 — tui: handle terminal thread-read fallback errors

- PR ref: `openai/codex#21045` (https://github.com/openai/codex/pull/21045)
- Head SHA: `661e9c9665a39297c1b8a8f159d8b344b6837181`
- Title: tui: handle terminal thread-read fallback errors
- Verdict: **merge-as-is**

## Review

This is a textbook narrow race-fix. The bug is the metadata-only fallback
`thread_read(thread_id, false)` at `codex-rs/tui/src/app/session_lifecycle.rs:235`
returning a terminal "thread not loaded" error instead of an empty-thread payload
when the picker entry survives but the app-server has already evicted the thread.
Mapping that one specific error class to the same `unavailable_live_thread_attach_error`
sentinel that the empty-turns branch already uses (lines 247-252) is the right call —
both states are observationally identical to the user (the thread is not yet
re-attachable) so they should produce one consistent message instead of leaking the
internal `thread/read failed during TUI session lookup` plumbing.

The `Err(err) => return Err(err)` arm at line 244 correctly preserves
non-terminal failures, so transient I/O / serde / network errors will still
propagate normally. That's the important guardrail — without it this would have
swallowed a class of real bugs.

Extracting `unavailable_live_thread_attach_error` as a shared helper at lines
106-110 also kills the duplicated `eyre!` literal between the two branches —
worth doing for consistency even if it weren't strictly required for the fix.

The cited test
`attach_live_thread_for_selection_rejects_empty_non_ephemeral_fallback_threads`
already covers the fallback rejection path, so reusing it for verification is
appropriate. Ship it.
