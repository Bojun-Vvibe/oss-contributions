# openai/codex #19776 — permissions: store thread sessions as profiles

- **Repo**: openai/codex
- **PR**: #19776
- **Author**: bolinfest
- **Head SHA**: 1db629a89874c7a9d41566ab460818f14790d5f4
- **Size**: +2 / −26 across four files (`tui/src/app/tests.rs`,
  `tui/src/app/thread_events.rs`, `tui/src/app/thread_session_state.rs`,
  `tui/src/app_server_session.rs`).
- **Stack position**: 5th in the bolinfest permissions stack
  (#19772 → #19773 → #19774 → #19775 → **#19776**).

## What it changes

Removes the legacy `sandbox_policy: SandboxPolicy` field from
`ThreadSessionState` (defined at `tui/src/app_server_session.rs:159` with the
explanatory comment "Legacy sandbox projection kept only for compatibility
fields that have not migrated to PermissionProfile yet") and updates every
read/write site so the cached TUI session state carries only the canonical
`permission_profile: PermissionProfile`.

Concrete edits:

- `app_server_session.rs:159-162`: deletes the field from the struct;
  `:1417-1423` deletes it from `thread_session_state_from_thread_response`'s
  pattern.
- `thread_session_state.rs:14-22`: drops the `let sandbox_policy =
  self.config.permissions.legacy_sandbox_policy(...)` call and the
  `session.sandbox_policy = sandbox_policy.clone()` assignment from
  `update_session`.
- `thread_session_state.rs:44-71`: drops the corresponding
  `active_legacy_sandbox_policy_for_cwd` calls from the thread-read path
  (both the initial-construct branch and the re-sync-on-cwd-change branch);
  `permission_profile` is still derived via
  `active_legacy_permission_profile_for_cwd(thread.cwd.as_path())`.
- `tests.rs` and `thread_events.rs` test fixtures lose all `sandbox_policy:
  SandboxPolicy::new_*_policy()` lines (six occurrences in `tests.rs`
  including `test_thread_session` itself; one in `thread_events.rs`).
- `tests.rs:2665` flips one assertion-side construction:
  `sandbox_policy: primary_session.sandbox_policy.clone()` becomes
  `sandbox_policy: SandboxPolicy::new_workspace_write_policy()` because
  the wire-protocol-side `SessionConfiguredEvent` still carries the legacy
  field (the policy stated in the PR body: "Leaves legacy `sandbox` fields
  in app-server request/response protocol paths unchanged; those are
  compatibility boundaries and are converted before entering cached TUI
  state").

## Strengths

- **Right scope at the right boundary.** The PR cleanly distinguishes "wire
  format compatibility" (kept) from "in-memory cached state" (collapsed onto
  `PermissionProfile`). The struct comment at the deleted field already
  warned that this projection was provisional; #19776 now closes that loop
  after #19772–#19775 made `PermissionProfile` carry the full story.
- **Net negative diff on a structural change.** −26 / +2 is the right shape:
  every deletion is a `sandbox_policy: …` field-init or assignment that the
  surrounding code already treats as a derivable shadow of
  `permission_profile`. There's no logic being added — only a redundant
  authority being removed.
- **Single behavior-changing edit is called out by its assertion site.**
  `tests.rs:2665` changes from
  `sandbox_policy: primary_session.sandbox_policy.clone()` to a direct
  `SandboxPolicy::new_workspace_write_policy()` literal because the cached
  side no longer has a `.sandbox_policy` field to clone. That's the right
  way to keep the wire-protocol fixture honest while letting the cache lose
  its mirror.
- **The diff preserves cwd-binding discipline.** The two surviving call
  sites at `thread_session_state.rs:51-69` still resolve the permission
  profile through `active_legacy_permission_profile_for_cwd(thread.cwd.as_path())`
  rather than a global session cwd, which is the same trap
  `thread_read_session_state_does_not_reuse_primary_permission_profile`
  (the test rewritten in #19773) names explicitly.
- **Test fixture deletions are mechanical and uniform.** Every removed
  `sandbox_policy:` line was paired with a `permission_profile:` line that
  already said the same thing in the canonical form, so the fixtures shrink
  without losing coverage.

## Concerns / nits

- **The PR body claims "old `sandbox` response values are still accepted, but
  they are immediately converted to a cwd-anchored profile."** That conversion
  isn't visible in this diff — it lives in the upstream
  `thread_session_state_from_thread_response` body that's only partially
  shown by the `-, sandbox_policy,` line at `app_server_session.rs:1420`.
  A reviewer with no context would have to trust the prose; a one-line
  PR-body cross-link to the `From<sandbox_policy>` conversion call site
  (e.g. `permission_profile.unwrap_or_else(|| PermissionProfile::from_legacy(&sandbox_policy, &cwd))`)
  would make the contract self-evident.
- **The `update_session` closure at `thread_session_state.rs:23-27`** still
  derives `permission_profile` via `chat_widget.config_ref().permissions...`
  rather than via the same `active_legacy_permission_profile_for_cwd(...)`
  helper used in the thread-read path. After this PR the chat-widget config
  is the only authority for the active session's profile, which is fine,
  but the asymmetry between the two derivation routes ("active session reads
  chat_widget" vs. "side-thread reads cwd-anchored helper") deserves a
  one-line comment near the closure so the next reader doesn't try to
  unify them and accidentally reintroduce the global-cwd-leak bug
  #19773's pin test was written to prevent.
- **No new pin test for the deletion itself.** The risk being eliminated
  ("future code reads `session.sandbox_policy` and gets a stale projection")
  is now type-system-enforced (the field is gone, so any reader fails to
  compile). That's the strongest possible guarantee, but the PR could call
  this out explicitly in the body — "removal is type-enforced; no runtime
  test required" — to forestall reviewers who would otherwise ask for one.
- **Stack ordering risk.** The PR depends on #19774 making
  `SessionConfigured` profile-only (so the wire format actually carries
  `permission_profile`), and on #19772/#19773 deriving config defaults and
  TUI thread state as profiles. If #19774 ships *after* this one merges,
  the wire-protocol-side `SessionConfiguredEvent` would still emit only
  `sandbox` and the conversion-at-ingestion guarantee from the PR body
  would silently fail. The Sapling stack footer makes the ordering
  explicit, but a release-note bullet ("merge after #19774") in the PR
  body would prevent a stack-out-of-order incident.

## Verdict

**merge-as-is.** Minimal, type-system-enforced removal of a redundant cached
authority that #19772–#19775 already replaced. Every deletion is mechanical,
the one behavior-side fixture flip is the right one, and the cwd-binding
discipline is preserved. The nits above are documentation/ordering polish
rather than correctness blockers — none change the diff. Ship after the
upstream stack lands.
