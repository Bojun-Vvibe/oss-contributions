# openai/codex#20277 — [codex] tighten exec safety parsing and sandbox path review

- PR: https://github.com/openai/codex/pull/20277
- HEAD: `f810207`
- Author: olliem-oai
- Files changed: 4 (+135 / -13)

## Summary

Two-pronged hardening of the exec-safety pipeline:
1. The bash tree-sitter parser now decodes backslash-escaped characters
   inside unquoted shell words, so `e\xec` no longer hides as a different
   `word` token from `is_known_safe_command` / prefix-rule matching.
2. A new `sandbox_writable_executable_path_requires_review` gate prevents
   auto-amendment proposals (and forces `Decision::Prompt`) when the
   command's program is an absolute (or multi-component) path that lives
   inside a workspace-writable region of a Restricted file-system policy
   under `RequireEscalated` permissions — i.e. running an executable the
   model itself could have just dropped on disk.

## Cited hunks

- `codex-rs/shell-command/src/bash.rs:152, 154, 162` — replaces three call
  sites that did `word_node.utf8_text(...).to_owned()` with a new
  `decode_unquoted_shell_word` helper.
- `codex-rs/shell-command/src/bash.rs:194-208` — `decode_unquoted_shell_word`
  iterates chars, treating `\` as escape: drops `\\\n` (line continuation),
  consumes the next char verbatim otherwise. This is the standard POSIX
  unquoted-word semantics and matches what `bash` itself does pre-execve.
- `codex-rs/shell-command/src/command_safety/is_safe_command.rs:472-479` —
  new regression test asserting
  `find /app -maxdepth 0 -e\xec sh -c '/usr/bin/su\do -n id' sh \;` is
  *not* known-safe. Pre-fix this slipped past because the literal `e\xec`
  word didn't match the canonical `-exec` token in any allow rule.
- `codex-rs/core/src/exec_policy.rs:253-262, 266-269, 282-294` — new
  `sandbox_writable_executable_requires_review` boolean folded into
  `auto_amendment_allowed`. When true, both the prefix-rule branch and the
  unmatched-command path skip generating a `requested_amendment`,
  preventing the model from being prompted to permanently allow what is
  effectively "run the file I just wrote."
- `codex-rs/core/src/exec_policy.rs:606-614` — same gate added at the
  entry of `render_decision_for_unmatched_command`, returning
  `Decision::Prompt` *before* the `is_known_safe_command` shortcut.
  Important: this means even a known-safe binary name (`cat`, `ls`) at a
  workspace-writable path falls through to user prompt, because path
  identity beats name identity for safety.
- `codex-rs/core/src/exec_policy.rs:707-727` — the gate itself: requires
  (a) `sandbox_permissions.requests_sandbox_override()`, (b) restricted
  policy kind, (c) policy lacks full disk write, and (d) program path is
  absolute *or* multi-component *and* writable under cwd via
  `can_write_path_with_cwd`. Bare names like `cat` short-circuit at
  step (d.path.components().count() <= 1).
- `codex-rs/core/src/exec_policy_tests.rs:1000-1023` — end-to-end test
  asserting `/tmp/cat /dev/null` under workspace-write +
  `RequireEscalated` returns `NeedsApproval { reason: None,
  proposed_execpolicy_amendment: None }`. The `None` amendment is the key
  invariant: the dangerous combo cannot be silently promoted.

## Risks

- `decode_unquoted_shell_word` collapses `\\\n` to nothing (correct for
  line continuation) but has no special handling for `\\\0` or other
  control-char escapes. Bash itself also only treats `\` as one-char
  escape, so behaviour matches; but worth a sanity test for `\\\\` →
  single backslash so the current contract is locked.
- The `program_path.components().count() <= 1` early-out at `:715-717`
  means a relative path like `foo/bar` (count=2) goes through the
  `can_write_path_with_cwd` check while bare `foo` (count=1) skips it.
  That's correct for PATH lookup vs. relative-from-cwd, but a bare program
  name that *also* happens to exist as a writable file at cwd-root
  (`./cat`) won't trigger the gate. Probably acceptable since
  `is_known_safe_command` still applies, but a follow-up could resolve
  bare-program lookups against cwd before the count check.
- `requests_sandbox_override()` gating means the new prompt only fires
  under escalated requests. A workspace-write run *without* escalation
  request still happily executes `/tmp/whatever` — that case is presumed
  safe because the sandbox itself would block writes outside the
  workspace, but it's worth a comment confirming that's the intended
  trust boundary.

## Verdict

**merge-as-is**

## Recommendation

Land it; both fixes target real bypass classes (parser-level escape evasion
and "drop-then-execute" amendment pollution) with focused tests pinning the
contract. Follow-up the bare-name-at-cwd-root edge in a separate PR if it
matters.
