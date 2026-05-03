# openai/codex PR #20849 — Make model_instructions_file paths context-relative

- Author: friel-openai
- Head SHA: `1f90775913466870192667d5f215922cc77788cd`
- State: DRAFT
- Diff: +95 / -8 across `codex-rs/core/src/config/config_loader_tests.rs` and `codex-rs/core/src/config/mod.rs`

## Observations

1. **`config_loader_tests.rs:1397-1430` adds `user_config_model_instructions_file_resolves_relative_to_codex_home`** — writes `model_instructions_file = "instructions.md"` to `$CODEX_HOME/config.toml`, places `instructions.md` in *both* `$CODEX_HOME` and `cwd`, then asserts `config.base_instructions == "user config instructions"` (the codex_home copy). This is exactly the regression the PR claims to lock in: a relative path in `config.toml` must anchor to `$CODEX_HOME`, not the user's working directory.
2. **`config_loader_tests.rs:1434-1474` adds `profile_model_instructions_file_resolves_relative_to_config_file`** — same pattern, but inside a `[profiles.relpath]` table. Asserts the profile-scoped path also resolves against the config file's directory. Two competing copies in `codex_home` and `cwd` make the test sharp; without that, a regression that silently picked up the cwd file would still pass.
3. **`config_loader_tests.rs:1480-1500` renames** the existing `cli_override_model_instructions_file_sets_base_instructions` to `..._resolves_relative_to_cwd` and changes the override value from an absolute `instructions_path.to_string_lossy()` to the relative literal `"instructions.md"` — proving CLI `-c` overrides anchor to cwd, not codex_home. This is the third leg of the "where does each path source resolve from" matrix and matches the comment update referenced in the PR summary.
4. **The runtime change in `config/mod.rs`** isn't visible in the truncated diff above, but the PR summary says it's a comment-only update to the loader, distinguishing config.toml paths from `-c` overrides. If that's literally the only `mod.rs` change, this PR is essentially "tests + docstring" — which is the right shape for locking down already-shipped behavior. Reviewer should grep for any silent code change in `mod.rs` before approving.
5. **One nit**: the three test names are a tight matrix (`user_config` / `profile` / `cli_override` × `_resolves_relative_to_*`). Worth a short comment block at the top of the section noting the matrix explicitly so future contributors don't accidentally remove one when refactoring.

## Verdict: `merge-after-nits`

Solid regression coverage with sharp differentiators (competing files in two directories) that catch the realistic failure modes. Pre-merge: confirm `mod.rs` is comment-only as described, and consider the matrix-block comment so the three tests are obviously a set.
