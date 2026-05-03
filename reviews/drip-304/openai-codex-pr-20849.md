# openai/codex #20849 — Make model_instructions_file paths context-relative

- **PR:** https://github.com/openai/codex/pull/20849
- **Head SHA:** `1f90775913466870192667d5f215922cc77788cd`
- **Author:** friel-openai
- **Size:** +95 / -8 across 2 files

## Summary

Two regression tests + a clarifying loader comment. Asserts that:
1. `model_instructions_file` declared in `$CODEX_HOME/config.toml` resolves relative to `$CODEX_HOME` (not cwd).
2. `model_instructions_file` declared inside a profile resolves relative to the config file's directory.
3. `-c` overrides keep their existing cwd-relative semantics — the loader comment is updated to make the distinction explicit.

## Specific references

- `codex-rs/core/src/config/config_loader_tests.rs:1395-1450` — new `user_config_model_instructions_file_resolves_relative_to_codex_home`. Sets up `$CODEX_HOME/config.toml` with a relative `instructions.md` path, also writes a *different* `instructions.md` under cwd, then asserts `config.base_instructions == "user config instructions"` (the codex-home one). This is exactly the right shape — without the codex-home anchor, the loader would've picked up the cwd file and the assertion would fail.
- `config_loader_tests.rs:1450+` — `profile_model_instructions_file_resolves_relative_to_config_file` test follows the same pattern for a profile-scoped declaration.
- `codex-rs/core/src/config/mod.rs` — comment on the `model_instructions_file` resolution path updated to call out the config.toml-vs-`-c` distinction. Useful future-maintainer note.

## Concerns

- None substantive. The tests are deterministic (use `tempdir()`), exercise the exact regression surface, and the comment cleanup reduces future confusion.
- Minor: the test names are long but accurate; matches the existing `config_loader_tests` naming convention.

## Verdict

**merge-as-is** — pure regression coverage + a clarifying comment. Zero behavior change in production code paths. Low-risk, high-value.
