# openai/codex #20085 — fix: don't auto approve `git -C ...`

- PR: https://github.com/openai/codex/pull/20085
- Head SHA: `e7fe8fe30da27542c84b018f4a325e28b99a7766`
- Files: `codex-rs/shell-command/src/command_safety/is_dangerous_command.rs`, `codex-rs/shell-command/src/command_safety/is_safe_command.rs`

## Citations

- `is_dangerous_command.rs:61` — `git_global_option_requires_prompt` exact-match arm extends `"-c" | "--config-env" | ...` to `"-C" | "-c" | "--config-env" | ...`.
- `is_dangerous_command.rs:70` — prefix arm extends `s.starts_with("-c") && s.len() > 2` to `(s.starts_with("-C") || s.starts_with("-c")) && s.len() > 2`, covering the joined form `git -C/path/to/repo branch ...`.
- `is_safe_command.rs:337` — pinning test changed: `["git","-C",".","branch","--show-current"]` was `is_known_safe_command == true`, now asserted `false`.
- `is_safe_command.rs:384-388` — new positive-prompt assertions: `["git","-C",".","status"]` and `["git","-C.","status"]` both NOT safe.
- `is_safe_command.rs:419-423` — bash-wrapper case: `["bash","-lc","git -C .project-deps/test-fixtures status"]` NOT safe (string-form passthrough also covered).
- `is_dangerous_command.rs:185-190` — new unit test `git_dash_c_requires_prompt` covering `-C` and `-C/path/to/repo`.

## Verdict

`merge-as-is`

## Reasoning

Tight, defensible security fix that closes a real escape from the auto-approve allowlist. `git -C <dir>` is functionally equivalent to `cd <dir> && git ...`, which means an auto-approved `git -C ../../some-other-repo branch -d main` happily mutates a repo the user almost certainly didn't mean to scope the approval to. The previous handling treated only `-c <key>=<value>` as warranting prompt (because `-c` lets you override hooks/aliases and execute arbitrary code through `core.gitProxy`, `safe.directory`, etc.), but missed `-C` even though it's the more common workspace-escape vector in agentic flows where the cwd is the project root.

The patch is structurally identical to the existing `-c` handling, which is the right pattern: same exact-match arm, same prefix-arm with the `&& s.len() > 2` guard so bare `-C` doesn't false-positive a different code path that splits the value off. Adding both at once keeps the two flags symmetrical, which matters because future `git_global_option_requires_prompt` edits will read both at the same time.

The test coverage is thorough in the right way: it pins (a) the positive case (`-C .` is now treated as dangerous), (b) the joined form (`-C.`), (c) the bash-wrapper passthrough (`bash -lc 'git -C ... status'`) which is the actually-exploitable surface in agentic shells where the model emits a one-line bash command, and (d) the original `git -c` cases stay green. The pinning-test flip (`branch --show-current` going from `true` to `false`) is conservative-correct: yes, that specific subcommand is read-only, but allowing `-C` for read-only subcommands while disallowing for write subcommands is a complexity nobody wants to maintain in a security boundary.

The PR description is one sentence ("It's safer to make sure these commands go through approval flows") which is appropriate given the test assertions are self-documenting. No regression risk — the worst case is one extra approval prompt for `git -C .` invocations that happen to be in the repo root, which a human will quickly approve once and which the always-allow list can pin if it becomes annoying.

Ship it.
