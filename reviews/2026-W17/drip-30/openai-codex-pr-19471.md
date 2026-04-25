# openai/codex#19471 — Use serial_test for Windows sandbox tests

**Verdict: merge-as-is**

- PR: https://github.com/openai/codex/pull/19471
- Author: dylan-hurd-oai
- Base SHA: `789f387982c51e8032766f91d4b026f4c50b0ff8`
- Head SHA: `861f47db33af8296d6a30d528f2a48def60043f0`
- +7 / -13

## Summary

Follow-up to #19453. That PR serialized the legacy Windows sandbox
process tests with a hand-rolled `Mutex<()>` + `MutexGuard` in
`windows-sandbox-rs/src/unified_exec/tests.rs`. This PR replaces that
bespoke mutex with `#[serial(windows_sandbox_legacy_process)]` from the
`serial_test` crate, which the crate already uses elsewhere via the
workspace dev-dependency. Net delta is -6 lines of synchronization
glue plus one named serial group annotation on each of the four legacy
process tests.

## Specific findings

- `codex-rs/windows-sandbox-rs/src/unified_exec/tests.rs:+4/-9`
  (head SHA `861f47db33af8296d6a30d528f2a48def60043f0`) — removes:
  ```
  static LEGACY_PROCESS_TEST_LOCK: Mutex<()> = Mutex::new(());
  fn legacy_process_test_guard() -> MutexGuard<'static, ()> {
      LEGACY_PROCESS_TEST_LOCK.lock().expect(...)
  }
  ```
  and the four `let _guard = legacy_process_test_guard();` call sites,
  in favor of `#[serial(windows_sandbox_legacy_process)]` on each
  `#[test]`. Named serial group is consistent across all four
  affected tests (`legacy_non_tty_cmd_emits_output`,
  `legacy_non_tty_powershell_emits_output`,
  `legacy_capture_powershell_emits_output`,
  `legacy_tty_powershell_emits_output_and_accepts_input`), so no test
  is accidentally orphaned out of the serialization group.
- `codex-rs/windows-sandbox-rs/Cargo.toml:+1` —
  `serial_test = { workspace = true }` is added under
  `[dev-dependencies]` (not a runtime dep, no shipping-binary impact).
- `codex-rs/Cargo.lock:+1` — single line added inserting `serial_test`
  into the resolved deps list for `codex-windows-sandbox`. Lock
  movement is minimal because the dep is already in the workspace.

## Rationale

This is exactly the right kind of follow-up: the previous PR's
maintainer review explicitly flagged the bespoke mutex as worse than
the crate-standard `serial_test` macro, and dc8f32f had used
`serial_test` originally. Restoring it brings the file back in line
with the rest of the codebase's serialization pattern, removes a
hand-rolled poison-handling path (`expect("legacy Windows sandbox
process test lock poisoned")`), and the named group ID
(`windows_sandbox_legacy_process`) gives `serial_test` enough
information to interleave these tests with others in the same crate
that don't share the legacy process resource — something the
`Mutex<()>` couldn't do because it held the lock across the whole
`#[test]` body. Test verification via
`cargo test -p codex-windows-sandbox` is the right scope. There's
nothing to nit; the dev-dependency add is the only Cargo.toml change,
no version pin (correct, takes the workspace pin), and the bazel
`just bazel-lock-update` / `just bazel-lock-check` invocations in
the PR body confirm the bazel-side lockfile was regenerated. Merge.
