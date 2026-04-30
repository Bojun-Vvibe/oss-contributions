# openai/codex #20406 — tui: hydrate thread permissions from profiles

- Head SHA: `56167305006fabdd01ab38cdcb5f3825cb9b348b`
- Files: `codex-rs/tui/src/app_server_session.rs`
- Size: +30 / -85

## What it does

`ThreadStartResponse`, `ThreadResumeResponse`, and `ThreadForkResponse` from
the app-server now all carry an explicit `permissionProfile` field. The TUI
response mappers were still doing a fallback dance: if `permissionProfile`
was present, use it; otherwise rebuild a `PermissionProfile` by calling
`PermissionProfile::from_legacy_sandbox_policy_for_cwd(&sandbox.to_core(), cwd)`
on the legacy `sandbox` field — but only when running in
`ThreadParamsMode::Remote`, i.e. against an old app-server.

This PR retires that legacy-fallback path:

- `permission_profile_from_thread_response` at
  `codex-rs/tui/src/app_server_session.rs:1413-1425` is collapsed from a
  five-arg function (sandbox, permission_profile, cwd, config,
  thread_params_mode) to a two-arg one (permission_profile, config).
  When `permission_profile` is `Some`, return it. Otherwise, return
  `config.permissions.permission_profile()` — i.e. the TUI's locally
  configured profile, not a reconstruction from `sandbox`.
- All three call sites (`thread_session_state_from_thread_start_response`
  at :1340, `…_resume_response` at :1356, `…_fork_response` at :1372) lose
  the `thread_params_mode` parameter.
- The three outer `started_thread_from_*_response` wrappers (at :1300,
  :1313, :1326) likewise drop `thread_params_mode` from their signatures
  and call sites in `start_thread`/`resume_thread`/`fork_thread` (lines
  :379, :404, :431).
- The test at :1913 is updated to call
  `started_thread_from_resume_response(response.clone(), &config)` with
  no third arg.

## What works

This is the correct direction. The legacy reconstruction was a one-way
projection: `PermissionProfile -> SandboxPolicy -> PermissionProfile`
necessarily lost any profile field that didn't fit in `SandboxPolicy`
(deny-read carve-outs, multi-glob scans, etc.). With the app-server now
sending `permissionProfile` natively, falling back to the user's
configured profile is strictly safer than rebuilding from a lossy field —
the worst case is "the TUI shows the same profile it had before connecting"
which is what most users would expect anyway, instead of "the TUI silently
shows a lossy reinterpretation of what the server thinks is in effect".

The function signature shrinkage (5→2 args, 6→3 args at the wrapper
layer) is a genuine simplification and removes the ambient
`thread_params_mode` flag from a code path that no longer needs to
distinguish.

## Concerns

1. **Old app-server skew.** The PR comment at :1421-1423 explicitly
   acknowledges that if `permission_profile` is `None`, the TUI now
   shows the user's locally configured profile rather than what the
   server thinks. For users connecting a new TUI to an old app-server
   that doesn't yet send `permissionProfile`, this is silently wrong —
   the server may be enforcing read-only while the TUI displays
   `workspace-write`. Two mitigations worth considering:
   - Log a `tracing::warn!` once per session when the fallback is taken,
     so the user can see "your TUI is older than the app-server it's
     connected to, permissions display may be stale".
   - Add a one-line check on connect: if the protocol version reported
     by the server is < the version that introduced `permissionProfile`,
     refuse to connect (or at least emit a top-bar banner).
2. **`ThreadParamsMode` now potentially unused.** The diff removes the
   parameter from these three call paths but doesn't show whether
   `ThreadParamsMode::Remote` still has any callers. If it doesn't,
   the enum is dead code; if it does, this PR has narrowed its
   meaning silently. Worth a follow-up grep to either delete the
   variant or document what it now signals.
3. **Test coverage gap.** The diff updates the existing resume test
   to drop the `ThreadParamsMode::Remote` arg but doesn't add a test
   for the new "permission_profile is None → fall back to config"
   branch. That branch is the riskiest part of this change (it's the
   one that silently disagrees with the server). A test that
   constructs a response with `permission_profile: None` and asserts
   the resulting `ThreadSessionState` carries `config.permissions.permission_profile()`
   would close the loop.

Verdict: merge-after-nits

## What I learned

When the wire format gains a higher-fidelity field that supersedes a
lossy one, the right pattern in the consumer is "use the new field if
present, fall back to *your own configured state* if absent" — never
"reconstruct the new field from the lossy one". Reconstruction looks
defensive but is actively harmful: it manufactures a confident-looking
but wrong answer. Falling back to local config at least makes the
disagreement visible (the TUI shows what *the user* configured, not
what the server thinks), and a one-line warn-on-fallback turns the
silent skew into a debuggable one.
