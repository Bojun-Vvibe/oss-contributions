# openai/codex PR #20679 — Detect implicit skill reads with parsed commands

- Head SHA: `26c058dd2ec91abd83b5eacf0ed0ad92600aae80`
- Size: +81 / -14, 5 files

## Specific refs

- `codex-rs/core-skills/src/invocation_utils.rs:103-117` — replaces the hard-coded reader allowlist (`cat sed head tail less more bat awk`) with a scan over `parse_command_impl(...)` filtering for `ParsedCommand::Read { path, .. }`. Now `nl SKILL.md` and any other command the shell parser classifies as a read is detected; pipelines like `cat SKILL.md | head` keep their partial-read signal.
- `codex-rs/core-skills/src/invocation_utils.rs:120-127` — new `tokens_with_command_basename` helper normalizes the program token via `command_basename(...).to_ascii_lowercase()` *before* feeding tokens to the parser, so `/usr/bin/CAT` and `cat` resolve identically — important since `parse_command_impl` is basename-aware.
- `codex-rs/core-skills/Cargo.toml:25` + `codex-rs/Cargo.lock:2587` — adds `codex-shell-command` as workspace dep; clean addition, no version pin drift.
- `codex-rs/core-skills/src/invocation_utils_tests.rs:78-98` — `skill_doc_read_detection_uses_parsed_read_commands` pins the `nl SKILL.md` regression case.
- `codex-rs/core/tests/suite/skills.rs:154-194` — `parsed_skill_doc_reads_detect_loaded_repo_skill` end-to-end test through `ThreadManager` + real skills load.

## Assessment

Net deletion of bespoke detection logic in favor of the canonical shell parser is the right direction — keeps skill detection in lockstep with how the rest of the agent classifies commands. The basename normalization on the program-only token (rest of args left untouched) is correct: `parse_command_impl` keys on basename, args are passed through as-is. One minor concern: the new code drops the explicit `token.starts_with('-')` flag-skip — but that filtering now lives inside `parse_command_impl`, which is the right layering. Test coverage adds both unit and integration. `cargo test` and `just bazel-lock-check` cited as passing.

verdict: merge-as-is
