# openai/codex #20849 — Test model_instructions_file context-relative paths

- PR: https://github.com/openai/codex/pull/20849
- Author: friel-openai
- Head SHA: `1f90775913466870192667d5f215922cc77788cd`
- Updated: 2026-05-03T09:41:14Z

## Summary
Adds three focused tokio integration tests in `codex-rs/core/src/config/config_loader_tests.rs` that pin down the resolution rules for `model_instructions_file` after a recent refactor: (1) `user_config_model_instructions_file_resolves_relative_to_codex_home` confirms a relative path in `~/.codex/config.toml` resolves against `codex_home`, not cwd; (2) `profile_model_instructions_file_resolves_relative_to_config_file` confirms the same for `[profiles.X]` blocks; (3) `cli_override_model_instructions_file_resolves_relative_to_cwd` rewrites the previous `cli_override_model_instructions_file_sets_base_instructions` to assert that CLI-supplied relative paths resolve against effective cwd. Also adds a 3-line clarifying comment in `codex-rs/core/src/config/mod.rs:2680-2684` documenting that the config layer loader is now responsible for resolving these relative paths.

## Observations
- `codex-rs/core/src/config/config_loader_tests.rs:1397-1432` (test 1): writes `instructions.md` in *both* `codex_home` and `cwd` with different content, asserts the loaded value is the `codex_home` one. Strong test — explicitly proves the resolution is not cwd-relative. The two-file setup is exactly what catches a regression where someone "fixes" the resolution by basing it on cwd.
- `codex-rs/core/src/config/config_loader_tests.rs:1435-1480` (test 2): same shape for `[profiles.relpath].model_instructions_file = "profile-instructions.md"`. The two-file decoy ensures we'd catch a regression that resolves against cwd. Selector via `config_profile: Some("relpath".to_string())` is the right harness path.
- `codex-rs/core/src/config/config_loader_tests.rs:1483-1499` (test 3): the CLI-override test was rewritten to use a *relative* `instructions.md` (instead of the prior absolute path) and to plant decoys in `codex_home` and `cwd`. The asserted content is `"cli override instructions"` from `cwd/instructions.md`, locking in cwd-relative behavior for CLI overrides. Critical because this is the one path where the resolution rule differs from config.toml entries.
- `codex-rs/core/src/config/mod.rs:2680-2684`: the comment now reads "Relative paths have already been resolved by the config layer loader: config.toml values resolve against the config file directory, while CLI overrides resolve against the effective cwd." Good documentation. The deleted "If the path is relative, resolve it against the effective cwd" comment was stale and misleading.
- The PR is test-only plus a comment update. No production logic changed in `mod.rs:2682-2690`. That's the right scope for a "lock in current behavior" PR.
- Nit: the renamed test `cli_override_model_instructions_file_resolves_relative_to_cwd` is clearer than the old `..._sets_base_instructions` name. Worth scanning for any external CI configs or `cargo test` invocations that targeted the old name; unlikely but a 30s grep.
- Consider adding a fourth test for `~`-expansion in `model_instructions_file` (does it tilde-expand? does it stay literal?). Not blocking; arguably a separate concern.

## Verdict
`merge-as-is`
