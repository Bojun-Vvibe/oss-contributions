# openai/codex PR #20469 — windows-sandbox: isolate allow path computation from legacy policy

- Repo: `openai/codex`
- PR: https://github.com/openai/codex/pull/20469
- Head SHA: `f6c34628bd6a57548a3c2fa8fc5729ba9eb47b18`
- State: OPEN
- Files: 1 (`codex-rs/windows-sandbox-rs/src/allow.rs`), ~+90/−~80
- Verdict: **merge-as-is**

## What it does

Lifts the `SandboxPolicy::WorkspaceWrite { writable_roots,
exclude_tmpdir_env_var, .. }` destructure out of the body of
`compute_allow_paths(policy, ...)` and into a new top-level `let-else`
that immediately returns `AllowDenyPaths::default()` for any
non-`WorkspaceWrite` policy. The body proper now operates on a typed
`WorkspaceWriteAllowPathInputs` struct (5 fields, all borrowed) via a
new private `fn compute_workspace_write_allow_paths(input)`; the public
entry point becomes a thin adapter.

## What's right

- This is the same Sapling-stack pattern that drip-211 reviewed for
  codex #20455 / #20456: relocate the projection from a downstream call
  site into the producer so the "is this WorkspaceWrite?" decision lives
  at one named, typed boundary instead of three nested
  `matches!(policy, SandboxPolicy::WorkspaceWrite { .. })` /
  `if let SandboxPolicy::WorkspaceWrite { writable_roots, .. } = policy`
  checks scattered through the function body.
- Before, the function had three separate guards on the same enum
  variant:
  - `let include_tmp_env_vars = matches!(policy, SandboxPolicy::WorkspaceWrite { exclude_tmpdir_env_var: false, .. });`
  - `if matches!(policy, SandboxPolicy::WorkspaceWrite { .. }) { ...add_writable_root(command_cwd, ...)... }`
  - `if let SandboxPolicy::WorkspaceWrite { writable_roots, .. } = policy { for root in writable_roots { ... } }`

  All three now collapse to a single `let-else` at the top, and the body
  is structurally guaranteed to be running with a known-extracted
  `writable_roots: &[AbsolutePathBuf]` and `include_tmp_env_vars: bool`.
- The new struct `WorkspaceWriteAllowPathInputs<'a>` borrows everything
  (no clones), and the adapter passes `writable_roots` as
  `&[AbsolutePathBuf]` instead of going through the previous
  `root.clone().into()` path conversion at the call site — a small
  allocation reduction on a hot input.
- All five existing tests are kept and rewritten through a new test-only
  helper `workspace_paths(policy_cwd, command_cwd, env_map,
  writable_roots, include_tmp_env_vars)` that goes directly through the
  new private function. This is the right shape for "the public
  function is now a degenerate adapter, but the test surface is the
  computation, not the adapter": tests target the actual logic and stop
  re-asserting the destructure pattern.
- Behavior is byte-identical for `WorkspaceWrite`. For non-`WorkspaceWrite`
  variants, the prior code returned an empty `AllowDenyPaths` after
  running zero `add_writable_root` calls (because the outer `if matches!`
  was false); the new `let-else` returns `AllowDenyPaths::default()`,
  which is the same empty value. The `include_tmp_env_vars` arm at the
  end was *also* gated behind no policy check before, meaning a
  hypothetical `DangerFullAccess` etc. would have walked the TEMP/TMP
  env vars and added them to allow — the new code skips that for
  non-`WorkspaceWrite`. That's a subtle behavior change for non-WS
  policies, but it's correct (the tmpdir allowance was conceptually a
  property of WS, not of all policies).

## Notes

- One subtle behavior change worth flagging in the PR description: the
  old code's `include_tmp_env_vars` block at the bottom of the function
  ran for *all* policy variants (it was outside the `if matches!`
  guard). The new code's early-return for non-WS short-circuits before
  the tmp arm, so e.g. a `ReadOnly` policy (or any future variant) that
  used to get TEMP/TMP injected into the allow list now gets an empty
  `AllowDenyPaths`. In practice this matters only if any caller invokes
  `compute_allow_paths` with a non-`WorkspaceWrite` policy and then uses
  the resulting allow list, which would be a bug at the call site
  anyway — but it's a behavioral delta the PR description doesn't call
  out.
- The struct lives in the same file (private to the module). Right
  scope.
- Function naming is asymmetric: the public function is
  `compute_allow_paths`, the new private one is
  `compute_workspace_write_allow_paths`. The naming forces honesty
  about scope — good — but the public name no longer mentions WS even
  though it now early-returns for everything else. A future grep for
  "what computes allow paths for a non-WS policy?" will land on this
  function and discover the answer is "nothing." Worth a one-line
  doc-comment on the public function.

## Nits (very minor, not blocking)

1. Doc-comment the public entry point: "Returns an empty
   `AllowDenyPaths` for any policy variant other than `WorkspaceWrite`;
   non-WS policies do not contribute writable roots through this path."
   That documents both the early return and the tmpdir behavior delta.
2. The test helper `workspace_paths` is one indirection layer above the
   call site; given that all 5 tests use it identically, consider
   inlining it, or — better — use `#[rstest]` or a small table to
   exercise the {protected_subdir = .git, .codex, .agents} cases as one
   parameterized test instead of three near-clone test bodies.
3. PR title says "isolate allow path computation from legacy policy" —
   "legacy" suggests the pattern relates to the broader PR series
   draining `SandboxPolicy` projection out of call sites (cf. PR
   #20455, #20456 from drip-211). A one-line back-reference in the PR
   body would help the reviewer place this PR on the stack.

## Why merge-as-is

Same shape as the codex #20455/#20456 stack, applied at the
windows-sandbox boundary. The diff is a faithful refactor — every test
still passes the same assertion against the same (now privately
addressed) computation, and the public API is unchanged. The minor
behavior delta for non-WS policies is correct (and arguably a
latent-bug fix). Nothing here needs to land before merge.
