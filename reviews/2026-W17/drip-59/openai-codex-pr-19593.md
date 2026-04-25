# openai/codex #19593 — test: isolate remote thread store regression from plugin warmups

- URL: https://github.com/openai/codex/pull/19593
- Head SHA: `6fe18bbbff0a3ec2b6f8b9acd6bd2ad1703f9cb8`
- State: MERGED
- Files: `codex-rs/app-server/tests/suite/v2/remote_thread_store.rs` (+5/-0)
- Verdict: **merge-as-is**

## What it does

Closes a Windows flake in
`thread_start_with_non_local_thread_store_does_not_create_local_persistence`
(follow-up to #19266). The assertion compares the post-run codex_home
listing against a frozen baseline, and BuildBuddy started showing an
unexpected top-level `.tmp` entry created by plugin startup warmups —
plugin-side noise rather than thread-store leakage.

## Diff notes

Two surgical changes in `remote_thread_store.rs`:

1. `:54-58` — a comment in front of `create_config_toml_with_thread_store`
   explaining *why* plugins are now disabled in the fixture (so the
   regression keeps signalling on its actual concern: thread persistence
   artifacts). Important — anyone reading this test in 6 months would
   otherwise wonder why plugins are off.

2. `:248-254` — the inline config TOML gets a new section:

   ```
   [features]
   plugins = false
   ```

   This is the right knob — `[features]` is the existing surface for
   toggling app-server feature flags during tests, and disabling
   plugins here is scoped to this one fixture, so other tests
   (which presumably *want* plugin coverage) keep their behaviour.

## Risk surface

- The fix narrows what this test guards (it no longer notices if the
  *plugins* feature accidentally starts writing to codex_home).
  That's fine — there's a separate plugin warmup test surface for that.
- Reviewer might want a mirrored regression test specifically for the
  plugin path so the warmup-to-`.tmp` interaction stays observable.
  Not blocking; this PR is just unflaking CI.

## Why this verdict

Five-line targeted fix with an explanatory comment. Lands clean. The
trade-off (narrower regression scope) is documented and reasonable.
