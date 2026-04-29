# Review: openai/codex#20113 — fix(exec_policy) heredoc parsing file_redirect

- **Author**: dylan-hurd-oai
- **Head SHA**: `be8e4e41dee25d60ed5f68092790fbb5bac93ada`
- **Diff**: +106 / −8 across 3 files (`codex-rs/core/src/exec_policy.rs`, `codex-rs/core/src/exec_policy_tests.rs`, `codex-rs/shell-command/src/bash.rs`)
- **Draft**: no

## What it does

Closes a sandbox-bypass primitive introduced by #10941. That earlier PR taught `parse_shell_lc_single_command_prefix` to walk through heredoc-attachment tree-sitter nodes (`heredoc_redirect`, `herestring_redirect`, **`file_redirect`**, `redirected_statement`) so that a command like `python3 <<'PY' ... PY` would be recognized as having executable prefix `python3` and could be matched by prefix rules. The bug: bundling `file_redirect` into the same allowlist meant that `cat <<'EOF' > /some/important/folder/test.txt\nhello world\nEOF` *also* collapsed to prefix `cat`, and a `prefix_rule(pattern=["cat"], decision="allow")` then approved a write to an arbitrary absolute path outside the sandbox.

The fix is the right shape: it splits the two semantics rather than narrowing one. In `shell-command/src/bash.rs`:

- `parse_shell_lc_single_command_prefix` now early-returns `None` when the parsed root contains a `file_redirect` named descendant (`bash.rs:131-138`), so the command no longer collapses to its executable prefix and the policy can't approve it via `prefix_rule`.
- The existing `is_allowed_heredoc_attachment_kind` allowlist drops `"file_redirect"` (`bash.rs:236-262`) — the second line of defense — with a refreshed comment explaining the security/semantics distinction: heredocs attach stdin, file redirects can write outside the sandbox.
- A new sibling helper `shell_lc_contains_file_redirect(command: &[String]) -> bool` (`bash.rs:142-150`) probes the same tree and returns `true` only when the parse succeeds without errors *and* a `file_redirect` is present.

In `core/src/exec_policy.rs:709-714`, `commands_for_exec_policy` is extended: after trying the existing `parse_shell_lc_single_command_prefix` path, it consults `shell_lc_contains_file_redirect(command)` and, when true, returns `(vec![command.to_vec()], true)` — i.e., it forces the *full* command vec into policy evaluation (preserving the redirect target so any path-aware rule can see it) and sets the `is_collapsed` second-tuple-slot to `true`, which downstream signals "treat this as a single command for approval" without the prefix-rule shortcut.

The test surface is the bug-class-driver here:

- `exec_policy_tests.rs:740-762` (`forbidden_prefix_rule_still_applies_to_heredoc_script`) — heredoc-with-no-redirect to `python3` is still forbidden by `prefix_rule(pattern=["python3"], decision="forbidden")`, with the literal expected reason string `"\`bash -lc \"python3 <<'PY'\nprint('hello')\nPY\"\` rejected: policy forbids commands starting with \`python3\`"`. Pins the regression-side: the original heredoc-prefix-extraction path is still alive for non-file-redirect cases.
- `exec_policy_tests.rs:817-839` (`heredoc_redirect_without_escalation_runs_inside_sandbox`) — `cat <<'EOF' > /some/important/folder/test.txt\n...\nEOF` under `SandboxPermissions::UseDefault` returns `ExecApprovalRequirement::Skip { bypass_sandbox: false, proposed_execpolicy_amendment: None }` — i.e., the sandbox catches the write attempt at runtime instead of pre-approving it through the prefix rule.
- `exec_policy_tests.rs:842-864` (`heredoc_redirect_with_escalation_requires_approval`) — same command under `SandboxPermissions::RequireEscalated` plus an explicit `prefix_rule(pattern=["cat"], decision="allow")` returns `ExecApprovalRequirement::NeedsApproval { reason: None, proposed_execpolicy_amendment: None }`, which is the escalation-required leg. This is the scenario where the bug was most dangerous: a user-allowed `cat` rule could write anywhere.
- `bash.rs:551-572` adds a unit-level inversion of an existing test: previously `parse_shell_lc_single_command_prefix_accepts_heredoc_with_extra_redirect` asserted `Some(vec!["python3".to_string()])`, now `..._rejects_heredoc_with_extra_file_redirect` asserts `None`. Plus a new `shell_lc_contains_file_redirect_detects_heredoc_with_extra_file_redirect` confirming the new helper recognizes a `cat <<'EOF' > ~/Pictures/sandbox_test.txt` pattern.

## Concerns

1. **The reason-string expected by `forbidden_prefix_rule_still_applies_to_heredoc_script` is fragile.** It pins the exact rendering `"\`bash -lc \"python3 <<'PY'\nprint('hello')\nPY\"\` rejected: policy forbids commands starting with \`python3\`"` including the embedded `\n` literals. Any cosmetic change to the reason formatter (escape rules, quoting) snaps this. Worth a `contains("python3")` + `contains("forbidden")` softer assertion, or at least a comment that this is a deliberately-frozen literal.

2. **`shell_lc_contains_file_redirect` has a `!root.has_error()` guard but the prefix path doesn't explicitly mirror it.** The early-return added at `bash.rs:131-134` runs `has_named_descendant_kind(root, "file_redirect")` *before* checking parse health, so a partially-broken parse could in principle false-positive-detect a file_redirect node and reject what would otherwise have been a benign heredoc. Inverting the order or adding `if root.has_error() { return None; }` at the top of the branch would tighten this.

3. **Append redirects (`>>`), here-string into command (`<<<`), and process substitution (`<(...)`) are not in the test matrix.** The named tree-sitter kind `"file_redirect"` is supposed to cover `>` and `>>`, but the test only exercises `>`. A `cat <<'EOF' >> /etc/passwd_appender ...` case would confirm the append variant lands in the same bucket. `herestring_redirect` is still in the allowlist (line 257), which is right (`<<<` doesn't write a file), but a positive-control test would prevent future allowlist drift.

4. **`commands_for_exec_policy` returning `(vec![command.to_vec()], true)` — what does the second `true` mean to downstream consumers?** The existing single-command-prefix branch above returns `(vec![single_command], true)`, so they're shaped consistently. But "is_collapsed = true" semantically meant "this is a single command, you can apply prefix rules." Now it also means "this is a single command, but you must NOT apply the prefix-rule shortcut." If any downstream consumer of that boolean was relying on the old conjoined meaning, it'll silently misbehave. A type-level change (e.g. `enum CollapseShape { PrefixMatchable, FullCommandOnly, Multi }`) would be more honest, but at minimum a comment near `exec_policy.rs:711-712` documenting the asymmetry would help.

## Verdict

**merge-after-nits** — exact-shape sandbox-bypass-class fix with a structural distinction between "heredoc that attaches stdin" and "heredoc that redirects stdout to a file" enforced at the parser layer and re-enforced at the policy layer. The boolean overload in `commands_for_exec_policy`'s return tuple is the load-bearing thing to clean up before merge; the brittle reason-string and the missing `>>` test are easy follow-ups.

