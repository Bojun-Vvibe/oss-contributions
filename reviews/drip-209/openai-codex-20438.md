# openai/codex#20438 — session: localize legacy sandbox turn fallback

- PR: https://github.com/openai/codex/pull/20438
- Head SHA: `71f7d562cdf8b595b0fc7dc8fff97d2f663fc639`
- Author: bolinfest
- Files: 3 changed (`session/handlers.rs`, `session/mod.rs`, `session/session.rs`), +9 / −16

## Context

The codex session API exposed a single legacy helper —
`Session::permission_profile_from_legacy_sandbox_update(sandbox_policy, cwd)` —
whose only caller was the `Op::UserTurn` legacy-fallback path in
`handlers.rs`. The helper itself just acquired the session state lock and
forwarded to `SessionConfiguration::permission_profile_from_legacy_sandbox_update`.
Its presence on `Session` made a one-off compatibility shim look like a
general session API surface, inviting future callers that don't realize the
legacy SandboxPolicy → PermissionProfile conversion is supposed to be a
one-way rare-path bridge during the deprecation of `SandboxPolicy`.

This PR is one in a series (cf. #20436, #20429, #20428, #20420, #20405, #20407,
#20410, #20411, #20412, #20414 in the same author's queue) tightening the
codex internal surface around the `PermissionProfile` migration: every place
that *can* speak `PermissionProfile` should, and the bridge code that still
takes a legacy `SandboxPolicy` should be a single, locatable, well-named
function.

## Design

**Move (`handlers.rs:323-336`).** The fallback conversion is now inlined into
the only caller, `permission_profile_with_legacy_fallback`:

```rust
match (permission_profile, sandbox_policy) {
    (Some(permission_profile), _) => Some(permission_profile),
    (None, Some(sandbox_policy)) => {
        let state = sess.state.lock().await;
        Some(
            state
                .session_configuration
                .permission_profile_from_legacy_sandbox_update(sandbox_policy, cwd),
        )
    }
    (None, None) => None,
}
```

The lock is acquired at the call site rather than inside the (now-removed)
session helper. Behavior is identical: the same lock is taken, the same
configuration method is called with the same arguments, the same
`PermissionProfile` is returned.

**Delete (`session/mod.rs:1356-1366`).** The `Session::permission_profile_from_legacy_sandbox_update`
method is removed entirely along with the `SandboxPolicy` import in `mod.rs`
that only existed to type its parameter. The import is correspondingly *added*
in `session/session.rs:5` because (per #20436 and other PRs in the series)
some other definitions in that file still reference `SandboxPolicy` for trace
metadata projection — so the import doesn't disappear from the crate, just
moves to where the remaining users live.

## Risk analysis

**Lock-acquisition reordering.** Previously, the lock was acquired *inside*
`permission_profile_from_legacy_sandbox_update`. Now it's acquired in
`permission_profile_with_legacy_fallback`. Are there any other locks held by
the caller chain? Walking up: `permission_profile_with_legacy_fallback` is
called from the `Op::UserTurn` handler in `handlers.rs`, which is itself
running inside the session loop's main task — no other session-state lock is
held at that point, so we can't deadlock against ourselves. The lock scope is
also narrower than before in the inlined version: previously the helper held
the lock for the full duration of `permission_profile_from_legacy_sandbox_update`'s
body; now the lock is held just long enough to call the configuration method
and drop the guard at the end of the `Some(...)` arm. Net change: identical
or slightly shorter critical section. ✅

**Caller surface.** `git grep permission_profile_from_legacy_sandbox_update`
needs to confirm that `Session::permission_profile_from_legacy_sandbox_update`
truly had only the one caller; the diff context shows the helper deletion is
clean (no unresolved references after the `mod.rs` change). The
`SessionConfiguration` method of the same name remains and is now called
directly. ✅

**Behavior preservation.** PR description explicitly claims "behavior intact:
when a turn omits `permission_profile` but includes legacy `sandbox_policy`,
the handler still derives a `PermissionProfile` from the current session
configuration and requested cwd." The diff matches that claim — the
projection function is the same `SessionConfiguration` method, the inputs are
the same `(sandbox_policy, cwd)` tuple, and the lock that guards "current
session configuration" is the same lock. ✅

## Verdict

**`merge-as-is`**

A clean −7 net LOC pure-refactor that makes the `Session` API surface honest
about which methods are general-use and which are legacy compatibility. The
key correctness move — keeping `state.lock().await` in the same logical
position so no other lock can sneak between `lock()` and the configuration
call — is preserved. No tests need to change because the externally observable
behavior of `permission_profile_with_legacy_fallback` is identical.

A useful follow-up would be to mark `permission_profile_with_legacy_fallback`
itself with `#[deprecated(note = "remove when Op::UserTurn drops sandbox_policy
field")]` or at minimum a `// LEGACY:` comment with the deprecation milestone,
so the next sweep of this code knows the function is itself slated for removal
once the wire schema can drop `sandbox_policy` entirely. Not a blocker for
this PR.

## What I learned

The "delete the one-call helper, inline at the call site" refactor is a
specific anti-API gesture: every method on a public-ish struct creates an
implicit invitation to call it from a second site. When the deprecation
plan says "this conversion will be removed within N releases," keeping it
behind a struct method makes future grep less effective at finding all the
callers (because the method name reads like a current-tense API). Inlining
into the *only* compatibility helper makes the call site itself the single
deletion point when the legacy field finally drops from the wire — exactly
the property a phased deprecation wants.
