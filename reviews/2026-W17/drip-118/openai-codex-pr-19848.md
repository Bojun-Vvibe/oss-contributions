# PR #19848 — Add preserved path shell preflight

- **Repo**: openai/codex
- **PR**: #19848 (DRAFT)
- **Head SHA**: `2b511ae3`
- **Author**: evawong-oai
- **Size**: +471 / -0 across many files (Cargo.lock, cli/, core/, shell-command/)
- **Verdict**: **needs-discussion**

## Summary

Third PR in a 5-PR stack (depends on #19846 policy primitive +
#19847 Seatbelt enforcement) that adds a *preflight* layer to
the agent shell handler and the `debug sandbox` CLI: before a
command is launched, parse it as a bash/zsh `-lc` script,
extract every literal redirection target and write-creating
command (`mkdir`, `touch`, `cp`, `mv`, etc.), and reject the
whole exec if any of those targets resolves under a "preserved
agent metadata path" (e.g. `.codex/`) per the new
`FileSystemSandboxPolicy`. The reviewer-focus note explicitly
calls this out as **UX only** — the actual security enforcement
lives in the macOS Seatbelt and Linux bubblewrap layers
(other PRs in the stack), so the preflight exists to give a
clean error message instead of letting users discover the
denial via a kernel-level "permission denied".

## Specific changes

- `codex-rs/cli/src/debug_sandbox.rs:148-153` — calls `codex_shell_command::preserved_path_write_forbidden_reason(&command, cwd, &policy)` before spawning the sandboxed child, and `anyhow::bail!`s with the returned reason string. Means `codex debug seatbelt`/`codex debug landlock` now refuse to even attempt the exec when the literal command writes to a preserved path.
- `codex-rs/cli/src/debug_sandbox.rs:255-549` — adds a Linux-only signal-forwarding subsystem: a `LINUX_SANDBOX_FORWARDED_SIGNALS` const for SIGHUP/SIGINT/SIGQUIT/SIGTERM, a `pre_exec` block on the spawned child that uses `prctl(PR_SET_PDEATHSIG, SIGTERM)` plus a `getppid() != parent_pid` race-close check, a `wait_for_debug_sandbox_child` that races `child.wait()` against a `recv_linux_sandbox_forwarded_signal` future via `tokio::select!`, and a `signal_debug_sandbox_child` helper that forwards the caught signal via `libc::kill` (treating ESRCH as success, which is the right idempotent shape). This is **substantial new unsafe code** for a PR titled "Add preserved path shell preflight" — the signal-forwarding feels orthogonal to the preflight goal.
- `codex-rs/core/src/tools/handlers/shell.rs:534-540` — wires the same `preserved_path_write_forbidden_reason` helper into the agent shell exec path: if the helper returns `Some(reason)`, the existing `exec_approval_requirement` is overridden to `ExecApprovalRequirement::Forbidden { reason }` so the user sees a "this command would write to a protected path" message rather than a generic sandbox denial.
- `codex-rs/shell-command/src/bash.rs:122-176` — new `parse_shell_lc_command_word_prefixes` helper that walks the tree-sitter bash AST and extracts argv-shaped prefixes from every command node, including ones nested in control flow or substitutions. Docstring is honest about the trade-off: "intentionally more permissive than `parse_shell_lc_plain_commands`... callers should use it only for conservative deny checks, not for allow-listing" — exactly the right shape for a preflight whose goal is "catch as many obvious literal writes as possible while admitting we'll miss exec/eval/parameter expansion".

## Risks

- **Scope creep, draft state**: PR is marked DRAFT, title is "Add preserved path shell preflight", but the diff includes a ~150-line Linux signal-forwarding subsystem (`block_linux_sandbox_forwarded_signals`, `recv_linux_sandbox_forwarded_signal`, `signal_debug_sandbox_child`, `wait_for_debug_sandbox_child`, the `pre_exec` block) that is not preflight code and not mentioned in the PR description. Either this subsystem belongs in a separate PR or the title needs to grow to mention it. As-is, a reviewer scanning for "preflight logic" would miss the signal-handling and the PR-stack walk-through doesn't explain why it's bundled here.
- **Preflight is admitted to be incomplete**: the docstring on `parse_shell_lc_command_word_prefixes` says the helper is "intentionally more permissive" and that callers should use it "only for conservative deny checks, not for allow-listing". That's correct framing but it means the preflight will still let through `bash -c "$(echo mkdir .codex)"`, parameter expansion that resolves to a preserved path at runtime, output redirection inside an `eval`, etc. The reviewer-focus note acknowledges this ("Direct commands such as `mkdir .codex` are intentionally not treated as preflight security logic here") but a user-visible message that says "blocked because this command writes to a protected path" while a one-character obfuscation slips through could create a false sense of security. A clearly-worded user-facing string like "your command appears to write to a protected path; if you believe this is incorrect, …" rather than "blocked: writes to protected path" would help.
- **`pre_exec` closure runs unsafe code in the child**: the SAFETY comment is present and correct ("only adjusts the child signal mask, installs a parent-death signal, and checks the inherited parent pid to close the fork/exec race"), but `prctl(PR_SET_PDEATHSIG, SIGTERM)` followed by an immediate `getppid() != parent_pid` check is a documented Linux idiom that's easy to get wrong if the closure ever grows. Worth pinning the invariant in a comment that future contributors must not allocate, lock, or call unrestricted libc inside this closure.
- **Signal forwarding only on Linux**: the macOS branch keeps `child.wait().await` directly without forwarding SIGHUP/SIGINT/etc. through a `tokio::signal::unix` watcher. If the goal is consistent UX across platforms, this is an asymmetry; if Linux-only is intentional (because seatbelt's kqueue handling already forwards), document it.
- **Test coverage**: the description says "Shell command tests passed locally" but the diff doesn't show any new tests in the shell-command crate exercising `parse_shell_lc_command_word_prefixes` or `preserved_path_write_forbidden_reason`. For a 471-line addition that gates exec, table-driven tests across `mkdir`, `touch >`, `cp`, `mv`, `tee`, redirections (`>`, `>>`, `&>`, `2>`), heredocs, and command substitution should be table stakes before exit-draft.

## Verdict

`needs-discussion` — the preflight is a defensible UX layer
(letting users see "blocked because protected path" rather than
a generic sandbox denial is real polish), and the framing in the
reviewer-focus note correctly punts security to the platform
sandbox layers. But the bundling of a substantial Linux signal-
forwarding subsystem under a "preflight" title is the kind of
hidden-scope that makes a 5-PR stack hard to land safely. Asks:
(1) split the Linux signal-forwarding into its own PR with its
own tests; (2) add a table-driven shell-parser test suite for
`parse_shell_lc_command_word_prefixes` covering at minimum the
write verbs and redirection forms named in the diff; (3) word
the user-facing rejection string so it doesn't promise
completeness the parser doesn't deliver; (4) document why the
signal forwarding is Linux-only.

## What I learned

A "preflight that's admittedly incomplete" is the hardest UX
shape to message correctly. If you say "blocked: writes to
protected path" and the user finds that `eval mkdir .codex`
slips through, they'll lose trust in every other denial message
the system ever shows them. The right framing for an honest-
imprecise checker is to make the message conditional on the
parser's confidence: "this command appears to write to a
protected path" (preflight, fast-fail) vs "this command
attempted to write to a protected path and was denied by the
sandbox" (post-exec, ground truth). The PR description gets the
architecture right (preflight = UX, sandbox = enforcement) but
the user-facing strings don't yet reflect the distinction.
