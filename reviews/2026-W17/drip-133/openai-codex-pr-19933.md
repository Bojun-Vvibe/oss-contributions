# openai/codex #19933 — Add `codex update` command

**Verdict:** `merge-after-nits`

- Repo: `openai/codex`
- PR: https://github.com/openai/codex/pull/19933
- Head SHA: `bd324edc`
- Addresses: #9274
- Diff: +66 / -2 across `README.md`, `codex-rs/cli/src/main.rs`, `codex-rs/cli/tests/marketplace_upgrade.rs`, `codex-rs/cli/tests/update.rs` (new), `codex-rs/tui/src/lib.rs`, `codex-rs/tui/src/update_action.rs`

## What it does

Adds a top-level `codex update` subcommand that detects the current install
channel (homebrew cask, npm, native binary, etc.) and runs the same
self-update action the TUI's `/update` prompt already uses. Previously,
typing `codex update` started an interactive session with `update` as the
first prompt, which is exactly the wrong rough edge for a user who has just
seen a "new version available" notice.

## Design analysis

Right shape. The reuse story is the load-bearing argument: the existing
`run_update_action(action)` function (already in `cli/src/main.rs:615`) and
the existing `get_update_action()` in `tui/src/update_action.rs` are both
doing the work. The new `run_update_command()` at `cli/src/main.rs:621` is
exactly the right thinness — it handles the two install-context outcomes:

```rust
let Some(action) = codex_tui::get_update_action() else {
    anyhow::bail!(
        "Could not detect the Codex installation method. Please update manually: https://developers.openai.com/codex/cli/"
    );
};
run_update_action(action)
```

The `cfg(debug_assertions)` arm bails with `"`codex update` is not available
in debug builds. Install a release build of Codex to use this command."` —
this matches the existing TUI `update_versions` gating
(`tui/src/lib.rs:172` is `#[cfg(any(not(debug_assertions), test))] mod
update_versions;`), so the new command is consistent with the rest of the
update infrastructure and won't surprise developers running from source.

The visibility change at `tui/src/update_action.rs:62` (`pub(crate) fn
get_update_action` → `pub fn get_update_action`) is gated behind
`#[cfg(not(debug_assertions))]`, and the matching re-export in
`tui/src/lib.rs:174-175` is also gated, so the public surface widens only
in release builds. That's a tighter API contract than just making it `pub`
unconditionally.

The dispatch site at `cli/src/main.rs:1023-1031` correctly routes through
`reject_remote_mode_for_subcommand` — `codex update` makes no sense in
remote mode where the binary lives on a different machine, and refusing it
up front is the right ergonomic call rather than running and confusing the
user about which install got updated.

The marketplace_upgrade.rs test edit at `:33` is unrelated-looking but
correct: adding a new `Update` enum variant changes clap's error message
for unknown subcommands from "unexpected argument 'upgrade'" to
"unrecognized subcommand 'upgrade'". The test was previously asserting on
the now-stale string. Fix-the-test-with-the-feature is correct here.

The new `cli/tests/update.rs` is debug-only (`#[cfg(debug_assertions)]`) and
asserts the bail message, which is the right test for what's actually
testable in CI without performing a real self-update.

## Risks

1. **Coverage gap on the release-build success path.** The only end-to-end
   test pins the debug-build "not available" branch. There's no test (even
   a mocked one) that asserts `run_update_action` actually fires when
   `get_update_action` returns `Some`. Mocking this is doable —
   `InstallContext::current()` could be parameterized — but the PR doesn't
   thread that. Acceptable because the underlying `run_update_action` already
   has its own tests where it lives, but worth a reviewer note.
2. **Two different sources of truth for "what update means."** This PR
   makes `codex update` and the TUI `/update` flow share `get_update_action`
   and `run_update_action`, which is good — but it means the user-visible
   string `"Could not detect the Codex installation method"` now exists in
   two places (the new `run_update_command` and the TUI's existing prompt
   path). If one drifts the other won't. Suggest extracting a shared helper.
3. **The README addition at `:31` is a single line.** No `--help` snippet,
   no flags documented (because there are no flags). Fine for this PR, but
   if `--check`, `--dry-run`, or channel-pinning flags land later, the README
   should grow with them.

## Suggestions

- Extract the not-detected error string into a `const` shared between
  `run_update_command` and the TUI's `update_prompt` module.
- Add a `--help` test asserting the new subcommand appears in the top-level
  help output. Otherwise `clap` regressions on the subcommand attribute go
  silent.
- Consider a future `codex update --check` that prints the available
  version without performing the update. The detection is already split
  from the action via `get_update_action()`, so it's a small follow-up.

## What I learned

The pattern here — "the TUI already had this; the CLI top-level just needs a
thin shim" — is the right way to add a subcommand for an action that was
previously accessible only through the interactive surface. The cost is
exactly the visibility-change ratchet (`pub(crate)` → gated `pub`) and one
new function. Resist the temptation to reimplement install-context detection
in the CLI: there's exactly one detection function, in exactly one module,
and it's release-build-only for a real reason (debug builds are not
self-updatable as a meaningful concept).
