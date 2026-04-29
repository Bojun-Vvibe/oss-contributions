# openai/codex #20113 — heredoc parsing file_redirect regression

- **PR:** https://github.com/openai/codex/pull/20113
- **Title:** `fix(exec_policy) heredoc parsing file_redirect`
- **Author:** dylan-hurd-oai
- **Head SHA:** f3e0a64efc0adba396672cf1167f04da6127a56d
- **Files changed:** 4 (`codex-rs/core/src/exec_policy.rs`,
  `codex-rs/core/src/exec_policy_tests.rs`, two `tests/suite/approvals.rs`
  scaffolding files), +296 / −18
- **Verdict:** `merge-after-nits`

## What it does

Fixes a regression introduced in #10941 where prefix-rule based
auto-amendments could be derived for heredoc commands containing file
redirects (e.g. `cat <<'EOF' > /important/path`). The original
`derive_requested_execpolicy_amendment_from_prefix_rule` was called
unconditionally at `exec_policy.rs:271-279`; this PR gates it behind the
already-computed `auto_amendment_allowed` flag:

```rust
let requested_amendment = if auto_amendment_allowed {
    derive_requested_execpolicy_amendment_from_prefix_rule(...)
} else {
    None
};
```

Then it adds 4 new scenario tests in `exec_policy_tests.rs:794-901`
covering:

1. heredoc fallback prompt that **does** match a prefix rule —
   amendment is suppressed (`drops_requested_amendment_for_heredoc_fallback_prompts_when_it_matches`)
2. heredoc with a `PATH=/tmp/evil:$PATH` env-var assignment in front of
   `cat` — must not collapse to the bare `cat` prefix rule
3. heredoc redirect without escalation runs inside sandbox (no auto-approve
   beyond sandbox)
4. heredoc redirect with `RequireEscalated` permissions correctly
   demands approval

Plus the two new `ActionKind::RunCommandWithPolicy` /
`RunCommandWithPrefixRule` variants in `tests/suite/approvals.rs:98-104`
to support the new scenarios in the existing approval matrix.

## What's good

- The fix is minimal and exactly localized: one branch on an already-
  computed flag. No restructuring of the surrounding decision logic.
- The new tests genuinely exercise the regression — test 2
  (`heredoc_with_variable_assignment_is_not_reduced_to_allowed_prefix`)
  is the most important one; without the fix, an attacker who can get
  the model to propose `PATH=/tmp/evil:$PATH cat <<'EOF' > /...` would
  have had Codex auto-amend the policy to allow that exact line.
- The new `ActionKind` variants in the approval matrix mean the same
  scenario now also runs through the integration suite, not just the
  unit test, which is the right place for "this rule shouldn't approve
  this command" guarantees to live long-term.
- `#[cfg(not(windows))]` correctly applied to the `PATH=...` test
  (line ~810) since Windows shells don't parse that as an env-var prefix.

## Nits / risks

- `policy_src(&self)` in `tests/suite/approvals.rs:124-136` exhaustively
  matches all `ActionKind` variants, but the list will need to be
  updated whenever a new `ActionKind` lands. Worth adding a
  `// when adding new ActionKind variants, decide whether to surface
  policy_src here` comment so future contributors don't silently default
  to `None` for a variant that should expose policy.
- Test name nit: `drops_requested_amendment_for_heredoc_fallback_prompts_when_it_matches`
  reads weirdly — "when it matches" (the prefix rule matches?) is the
  case where the bug used to fire. Consider
  `drops_requested_amendment_for_heredoc_when_prefix_rule_would_match`
  for clarity.
- No regression test added for the **non**-heredoc happy path that
  `auto_amendment_allowed=true` still derives an amendment — i.e. the
  fix is asymmetric; we have proof we suppress correctly, but no
  guard rail that we didn't *over*-suppress. Adding one positive case
  to the same module would close the loop.
- The PR body is one line ("Fixes a regression introduced in #10941").
  The commit will be the long-term reference for this CVE-class issue
  (sandbox escape via heredoc redirect). Worth expanding the body to
  explicitly call out the threat model so future blame digs surface
  the *why* without having to chase #10941.

## Verdict rationale

`merge-after-nits`: the fix itself is correct and well-tested; the nits
are about test naming, a defensive comment in the matrix scaffolding,
and PR-body provenance — none block merge but all are cheap to address
in this PR rather than in a follow-up.
