# openai/codex#20270 — [codex] Fix elevated Windows sandbox named-pipe access for Ninja

- PR: https://github.com/openai/codex/pull/20270
- HEAD: `de1e484`
- Author: iceweasel-oai
- Files changed: 3 (+92 / -10)

## Summary

The Windows elevated-sandbox code path was building restricted tokens
whose restricting-SID set didn't include the token user. Under Windows
ACL semantics that breaks any access to a securable object (named pipes
in Ninja's case) whose DACL grants the dedicated sandbox account but
none of the residual restricting SIDs (logon, Everyone, capabilities).
The PR introduces sibling factories
`create_{readonly,workspace_write}_token_with_caps_and_user_from` that
reads the base token's `TokenUser` SID via `GetTokenInformation` and
threads it into the restricting-SID list.

## Cited hunks

- `codex-rs/windows-sandbox-rs/src/token.rs:255-291` — new
  `get_user_sid_bytes(h_token)` helper. Two-call pattern
  (`needed=0` size query then real fetch), `read_unaligned` on the
  buffer (correct — `TOKEN_USER` from Win32 has alignment-1 layout),
  then `GetLengthSid` + `CopySid` into a fresh owning `Vec<u8>`. The
  ownership story is clean: caller gets bytes it controls and can pass
  to `create_token_with_caps_from` without lifetime entanglement with
  `user_buf`.
- `codex-rs/windows-sandbox-rs/src/token.rs:340-348, 352-355,
  356-368, 379-396` — four entry points refactored:
  `create_readonly_token_with_cap_from` and the two existing `_with_caps_from`
  variants now thread an extra `extra_restricting_sids: &[*mut c_void]`
  param (defaulted to `&[]`); two new `_with_caps_and_user_from`
  variants resolve the user SID and pass it as the lone extra.
- `codex-rs/windows-sandbox-rs/src/token.rs:411-420` — the SID-array
  build in `create_token_with_caps_from` is widened by
  `extra_restricting_sids.len()`; ordering is `Capabilities..., Extras...,
  Logon, Everyone`, with the `logon_idx`/`+1` indexing rebased off
  `extras_idx`. The comment at `:411` is updated to match the new exact
  order.
- `codex-rs/windows-sandbox-rs/src/elevated/command_runner_win.rs:31-32,
  244-249` — both `SandboxPolicy::ReadOnly` and `SandboxPolicy::WorkspaceWrite`
  branches switch from the original `_with_caps_from` factories to the
  new `_with_caps_and_user_from` variants. The non-elevated path (and
  the `DangerFullAccess` / `ExternalSandbox` branches) is untouched —
  this is exactly the surgical scope claimed in the doc comment "intended
  for the elevated sandbox backend, where the token user is the
  dedicated sandbox account."
- `codex-rs/windows-sandbox-rs/src/lib.rs:208-216` — re-exports the two
  new factory names alongside the existing ones, behind `#[cfg(target_os
  = "windows")]`.

## Risks

- `read_unaligned::<TOKEN_USER>(...)` reads through a raw pointer cast.
  `TOKEN_USER` is `repr(C)` with `{ User: SID_AND_ATTRIBUTES }` and the
  inner `Sid: PSID` is a pointer that points *into* `user_buf`.
  Subsequent `GetLengthSid(token_user.User.Sid)` and
  `CopySid(..., token_user.User.Sid)` both run *while `user_buf` is
  still alive* (it's the function-local `Vec<u8>`), so the pointer
  remains valid. Good — but if anyone later refactors to return the
  unaligned read while dropping `user_buf` early, they'll get UAF. A
  one-line comment at `:271` ("`token_user.User.Sid` aliases into
  `user_buf` — keep `user_buf` alive across the calls below") would
  prevent that.
- The original public API (`create_workspace_write_token_with_caps_from`
  / `create_readonly_token_with_caps_from`) is preserved. No deprecation
  marker. Downstream non-elevated callers that *should* also include the
  user SID won't get a compiler nudge — the doc comment makes it clear
  this is intentional, but the safety story for "when do you want
  `_with_user` vs not" deserves a paragraph in the module-level docs.
- `extra_restricting_sids` is a free-form `&[*mut c_void]` slice. Caller
  contract (each pointer must outlive the function call and point to a
  valid SID) is implicit. Adding `# Safety` notes on
  `create_token_with_caps_from` itself, or moving to a typed `&[Sid]`
  owning wrapper, would make future regressions noisier.
- No new tests — Windows token plumbing is hard to exercise outside an
  elevated process and the Ninja-named-pipe regression isn't easily
  unittable without a Windows runner. A small in-tree mock that asserts
  the SID array contains the user SID under the new entry points would
  at least catch list-construction regressions on a non-Windows host
  (currently behind `#[cfg(target_os = "windows")]`, so the ordering at
  `:411-420` isn't validated by CI's Linux fast-pass).

## Verdict

**merge-after-nits**

## Recommendation

Land after a one-line lifetime comment in `get_user_sid_bytes` and a
follow-up to either deprecate the old `_from` factories in elevated paths
or add a list-ordering test that runs on all OSes; the underlying ACL
semantics fix is correct and necessary for Ninja-style named-pipe
workloads.
