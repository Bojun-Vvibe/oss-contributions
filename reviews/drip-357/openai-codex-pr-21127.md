# openai/codex #21127 — fix(linux-sandbox): avoid panic on bwrap build failures

- **Repo**: openai/codex
- **PR**: [#21127](https://github.com/openai/codex/pull/21127)
- **Author**: viyatb-oai
- **Head SHA**: `830edd0da8d107471574fc65291cbd087184cf4d` (force-pushed; list initially showed `14dd038e05da7287b5a13cb56a0ce1800d89190d`)
- **Base branch**: `main`
- **Created**: 2026-05-05T02:20:05Z

## Verdict

`merge-as-is`

## Summary of change

Replaces a `panic!("error building bubblewrap command: {err:?}")` in the
linux-sandbox helper with a `Result`-based error path that exits with code 1
and a clean stderr message. The trigger case is bubblewrap legitimately
refusing read-only carveouts that cross a symlink the sandboxed process can
still rewrite — the sandbox is doing the right thing, but we were turning
"fail-closed deny" into a noisy panic with a Rust backtrace.

Touched surface:

- `linux_run_main.rs:336-337` and `:355-356` — both call sites of the helper
  now `.unwrap_or_else(|err| exit_with_bwrap_build_error(err))` instead of
  panicking inside the helper.
- `linux_run_main.rs:380-388` — `build_bwrap_argv` returns
  `CodexResult<crate::bwrap::BwrapArgs>`; the inner `create_bwrap_command_args`
  error is now propagated with `?` instead of `unwrap_or_else(|err| panic!...)`.
- `linux_run_main.rs:393-396` — new `exit_with_bwrap_build_error(err) -> !`
  prints `error building bubblewrap command: {err}` and exits 1.
- `linux_run_main.rs:449-462` — `preflight_proc_mount_support` and
  `build_preflight_bwrap_argv` are converted to return `CodexResult<...>`.
- `linux_run_main_tests.rs:64,108,177,196,272` — 5 unit tests get an
  `.expect("build bwrap argv")` (or "build preflight argv") tacked on, which
  preserves the panic-on-broken-test behavior in tests where panicking is fine.
- `tests/suite/landlock.rs:590-636` — new integration test
  `sandbox_reports_codex_symlink_build_failure_without_panicking` creates a
  `.codex` symlink pointing at a decoy dir inside a writable workspace root,
  asserts exit code 1, asserts stderr contains
  `"error building bubblewrap command: cannot enforce sandbox read-only path"`,
  and asserts stderr does NOT contain `"panicked at"`. Excellent regression
  coverage — both the positive (clean error) and negative (no panic) sides
  are pinned.

## What's good

- The fix is minimal, surgical, and the right shape: panic → propagate
  `CodexErr` → exit-with-message at the binary boundary, not in the helper.
- `exit_with_bwrap_build_error` is a single, dedicated function with a `!`
  return type — easy to grep, easy to extend later (e.g. JSON error output for
  the app-server channel).
- Switching `{err:?}` to `{err}` in the user-facing message is a quiet
  improvement (the `Display` impl on `CodexErr` is much more readable than the
  `Debug` formatting that the panic used).
- The new integration test exercises the *real* failure path (a `.codex`
  symlink replacement attack inside a writable root), not a synthetic mock,
  which is the right level of coverage for sandbox guarantees.
- The existing `sandbox_blocks_codex_symlink_replacement_attack` test on
  line 587 is preserved unchanged, so the security property is still asserted
  on the same input that previously panicked.

## Nits / questions

- The test asserts the literal string `"cannot enforce sandbox read-only path"`
  appears in stderr. That couples the test to the exact wording of the error
  in `create_bwrap_command_args`; if that message ever gets reworded, both
  tests need to be updated together. Optional: factor the error-message prefix
  into a shared `pub(crate) const` so the test imports the same constant.
- Exit code 1 is correct for "configuration / sandbox build failed", but worth
  confirming this doesn't collide with a sandbox-policy-deny exit code that
  callers (the parent codex process) already pattern-match on. A quick `grep
  -r "exit_code == 1" codex-rs/` would settle it; from the diff alone it's not
  visible.
- `exit_with_bwrap_build_error` writes to `eprintln!` — fine for the helper
  binary, but if this path is ever invoked from a subprocess whose stderr is
  parsed structurally, the consumer needs to know to expect a free-text line.
  Not a blocker, just worth noting in a follow-up.

## Risk

Low. The behavioral change is "panic → exit(1) with a clean message"; the
sandbox still fails closed. Test coverage explicitly pins both halves.

## Recommendation

Ship.
