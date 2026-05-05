# openai/codex PR #21089 — Fix `fork --last` cwd filtering

- Repo: `openai/codex`
- PR: https://github.com/openai/codex/pull/21089
- Head SHA: `638b8989c617`
- Size: +181 / -10 across 1 file (`codex-rs/tui/src/lib.rs`)
- Fixes upstream #20945.

## Summary

`fork --last` and `resume --last` are sister flows for picking the most
recent session, but they had drifted: `resume --last` (and remote-mode
`fork --last`) routed through `latest_session_cwd_filter`, which scopes
selection to the current cwd unless `--all` is passed; local-mode
`fork --last` did not, so it would silently fork a session from a
completely different directory. This PR replaces the conditional at
`codex-rs/tui/src/lib.rs:1281-1289` with an unconditional call to
`latest_session_cwd_filter(remote_mode, remote_cwd_override.as_deref(),
&config, cli.fork_show_all)`, which already handles all three cases
internally (local, `--all`, remote with override). Plus two real
end-to-end tests against on-disk rollout files.

## What I like

- The pre-fix code at lines 1281-1289 was the textbook "two flags fight
  each other" pattern: an outer `if remote_mode` and an inner
  `latest_session_cwd_filter(remote_mode, ...)`. That helper already
  encodes "local + show_all → None, local + !show_all → Some(cwd),
  remote → Some(remote_cwd_override or config.cwd)", so the outer
  conditional was both redundant and inverted for the local case. Just
  calling the helper is the right shape.
- The new `latest_session_cwd_filter_respects_scope_options` test at
  lib.rs:1885-1908 directly pins the helper's three-way truth table
  with named-comment positional args (`/*remote_mode*/ false,
  /*show_all*/ true`, etc.) — exactly the kind of test that would have
  caught this regression at introduction.
- The integration test
  `fork_last_filters_latest_session_by_cwd_unless_show_all` at
  lib.rs:1916+ writes real rollout JSONL files under
  `sessions/YYYY/MM/DD/rollout-…jsonl` with distinct cwds and asserts
  the picker only returns the in-cwd one without `--all` and the most
  recent overall with `--all`. That's the realistic invariant — pure
  unit tests on `latest_session_cwd_filter` alone could miss a wiring
  bug between picker and filter.
- `--all` semantics ("explicit opt-in for global latest-session") is
  preserved verbatim — the helper sees `show_all=true` and returns
  `None`, which the picker treats as "no cwd scope". So users who rely
  on `fork --last --all` are not broken.
- Remote-cwd override path is unchanged (still threads
  `remote_cwd_override.as_deref()` into the helper); the test asserts
  `remote_filter == Some(remote_cwd)` to lock that down.

## Nits / discussion

1. **Helper signature is now mandatory.** Before the patch, callers
   could in principle skip the helper and pass `None` directly. After
   this PR, every `--last` flow goes through it. That's the intended
   outcome but worth a short doc comment on
   `latest_session_cwd_filter` explaining "this is the canonical
   scope-resolver for `--last` flows; do not duplicate."

2. **Test verbosity.** `write_session_rollout` is defined inline in the
   test fn (lib.rs:1916+). It's clear, but the same helper would
   probably be useful for any future test that needs on-disk rollouts.
   Consider promoting to a `tests/common/` helper if other test files
   start needing it.

3. **No changelog / release-note hint.** Behavior change is small but
   user-visible: `fork --last` without `--all` is now cwd-scoped where
   previously it was not. Power users with cross-cwd workflows could
   notice. Worth a one-liner in release notes ("`fork --last` is now
   cwd-scoped by default; pass `--all` for previous behavior") so it
   doesn't surprise anyone.

4. **`unreachable!("app server should be initialized for --fork --last")`**
   at lib.rs:1290 is preserved from the old code. Fine, but if there's
   any path where `app_server` could legitimately be `None` (e.g.
   network init failure) this turns a recoverable error into a panic.
   Out of scope for this PR but flagging.

## Verdict

**merge-as-is.** Real bug fix, right shape (collapse a redundant
conditional, let the helper own the truth table), backed by a unit
test and an integration test against real rollout files. Behavior
change is intentional and aligns `fork --last` with `resume --last`,
which is what the issue asked for.
