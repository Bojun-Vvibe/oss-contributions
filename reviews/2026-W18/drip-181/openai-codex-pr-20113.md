# openai/codex#20113 — fix(exec_policy) heredoc parsing file_redirect

- **Repo**: openai/codex
- **PR**: [#20113](https://github.com/openai/codex/pull/20113)
- **Head SHA**: `f3e0a64efc0adba396672cf1167f04da6127a56d`
- **Author**: dylan-hurd-oai
- **Diff stats**: +296 / −18 (2+ files; one core source change, large test
  additions)

## What it does

Two related correctness fixes in `ExecPolicyManager` around how heredoc
shell commands interact with the prefix-rule "auto-amendment" feature:

1. The `derive_requested_execpolicy_amendment_from_prefix_rule` call at
   `codex-rs/core/src/exec_policy.rs:271-285` is now gated behind the
   existing `auto_amendment_allowed` flag. Previously the amendment was
   *derived* even when not allowed; the value was discarded downstream
   but the work — and any side-effects in matched-rule expansion — still
   happened.
2. New scenario coverage proves three real semantic cases that the
   previous logic got subtly wrong:
   - heredoc with **variable-assignment prefix**
     (`PATH=/tmp/evil:$PATH cat <<'EOF'`) must NOT be reduced to the
     allowed `cat` prefix rule (a forged-PATH attack would otherwise
     auto-allow).
   - heredoc with **file redirect** (`cat <<'EOF' > /some/file`) is
     allowed inside a workspace-write sandbox without escalation, but
     requires explicit approval when escalation is requested.
   - heredoc-only commands without a redirect or assignment should
     still auto-amend cleanly when matching a prefix rule.

## Code observations

- `codex-rs/core/src/exec_policy.rs:274-285` — the wrap is exactly:
  `if auto_amendment_allowed { derive_...(...) } else { None }`. Reads
  trivially. The implicit performance win (skipping the derive when
  amendment is disabled) is incidental; the semantic win is that any
  observable side-effect of the derive (e.g. logging, metrics, future
  cache mutation) no longer fires in disallowed-amendment paths.
- `codex-rs/core/src/exec_policy_tests.rs:794-812` —
  `drops_requested_amendment_for_heredoc_fallback_prompts_when_it_matches`
  exercises the case where the prefix rule actually fires AND the
  fallback prompt would normally produce an amendment. Asserts
  `proposed_execpolicy_amendment: None` because the heredoc path
  shouldn't produce an auto-amendment even when the underlying prefix
  rule matches. Good.
- `:817-838`
  `heredoc_with_variable_assignment_is_not_reduced_to_allowed_prefix` —
  this is the **load-bearing security test**. Without the fix, a policy
  that allows `cat` would let `PATH=/tmp/evil:$PATH cat <<'EOF'…EOF`
  silently slip through as "looks like cat", with the variable
  assignment masked by the heredoc framing. Test pins
  `ExecApprovalRequirement::Skip { proposed_execpolicy_amendment:
  Some(...full original command...) }` — i.e. the amendment is the
  *whole* command including the `PATH=` prefix, not the reduced `cat`,
  so any subsequent approval is informed.
- `:843-871` heredoc-with-file-redirect non-escalation case asserts
  `Skip { bypass_sandbox: false }` with the full command preserved in
  the amendment.
- `:875-901` heredoc-with-escalation case asserts `NeedsApproval` with
  the same full command. The pair of tests at `:843-871` and `:875-901`
  is the right shape: same syntactic command, behavior depends on
  `sandbox_permissions: SandboxPermissions::UseDefault` vs
  `RequireEscalated`.
- `codex-rs/core/tests/suite/approvals.rs:96-103,594-599,964+` — adds
  three new `ActionKind` variants (`RunCommandWithPolicy`,
  `RunCommandWithPrefixRule`, plus a new `Outcome::ExecApprovalWithAmendment`
  shape) and three matrix scenarios. The `policy_src(&self)` helper at
  `:124-137` exhaustively matches all 8 `ActionKind` variants, which is
  the right discipline — a new `ActionKind` will fail to compile until
  someone explicitly decides whether it carries a policy.

## Concerns

- **`Cfg-not-windows` gate at `:817`** — `heredoc_with_variable_assignment`
  is `#[cfg(not(windows))]`. The PR doesn't explain whether the underlying
  parser disagrees on Windows or whether the test scaffold can't drive
  `bash -lc` there. If the parser DOES handle this correctly on Windows,
  the test should run there too (security tests should not silently skip
  on a supported target).
- **Amendment is the full original command including the `PATH=` prefix**
  at `:830-834`. That's correct for "the human can see the full attempt",
  but if the human approves it without reading carefully, the malicious
  PATH still runs. The amendment surface should pair with a UI that
  highlights variable-assignment prefixes — this PR doesn't touch that
  but it's the most-likely follow-up.
- **No fuzz / property test** on heredoc-prefix combinations. The
  syntactic surface (here-doc delimiter quoting `'EOF'` vs `EOF` vs
  `EOF-`, here-string `<<<`, multi-redirect chains, command substitution
  `<(cmd)`) is rich and the prefix-rule reduction has a long history of
  edge cases. A small `proptest!` over (prefix-modifier × heredoc-shape ×
  redirect-target) would amortize across the next 10 of these PRs.

## Verdict: `merge-after-nits`

Strong fix, properly testified. Nits:

1. Either drop the `#[cfg(not(windows))]` gate at `:817` and run the
   test on Windows too, or document why bash-heredoc-variable-assignment
   parsing diverges on Windows in a comment. Skipping a security test
   on a supported platform is a smell.
2. Add a test asserting that a heredoc with **command substitution**
   `cat <<EOF; rm -rf /tmp/$(whoami) ; EOF` is similarly not reduced —
   the variable-assignment case is the most obvious bypass but
   `$(...)` and `` `...` `` injection inside a heredoc body that flows
   to a redirect target should also be covered.
3. Pair with a CLI/TUI follow-up that visually highlights
   variable-assignment prefixes in the approval prompt — this fix
   makes the right amendment available, but UX is what closes the
   loop.
4. Add a brief comment at `:274-285` explaining *why* the derive is
   gated on `auto_amendment_allowed` (not just the value-discard
   property) — namely that derive can have observable side-effects in
   matched-rule expansion. Otherwise a future refactor will inline
   it back inside the gate.
5. Cross-check that `assert_exec_approval_requirement_for_command`'s
   call site for the gpt-5.2 `model_override` in the
   `approvals.rs` scenarios is intentional vs. test scaffolding. The
   three new scenarios all pin `model_override: Some("gpt-5.2")` —
   if that's a default-test-baseline, fine; if it's a copy-paste from
   another scenario, may want to broaden across other model choices.

## Follow-ups

- Track a meta-issue "prefix-rule reduction safety surface" enumerating
  all known bypass categories: variable assignment, command
  substitution, glob expansion (`cat /etc/p* > /tmp/x`), `eval`
  wrapping, function redefinition, alias shadow, `IFS=` shenanigans.
- Consider a structured "shell script intent" parser that returns a
  typed `ParsedShellInvocation { prefix, env_assignments,
  redirects, substitutions, body }` so prefix-rule matching becomes a
  pure function on a typed value rather than string-shape heuristics.
