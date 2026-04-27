# openai/codex PR #19775 — `permissions`: derive `sandbox_policy` snapshot from canonical profile

- **PR**: https://github.com/openai/codex/pull/19775
- **Author**: @bolinfest
- **Head SHA**: `6c92cc40`
- **Size**: +26 / −16
- **Files**: `codex-rs/app-server/src/codex_message_processor.rs`, `codex-rs/core/src/codex_thread.rs`, `codex-rs/core/src/memories/tests.rs`

## Summary

Removes the `sandbox_policy` field from `ThreadConfigSnapshot` and replaces it with a `sandbox_policy()` *projection method* derived from `permission_profile` plus `cwd`. The motivation: the snapshot already carries `permission_profile` as the canonical permission state, and storing a parallel `sandbox_policy` made it possible for the legacy projection (consumed by app-server protocol fields, thread-metadata DB inserts, and resume-mismatch comparisons) to drift from the canonical profile when one was updated and the other wasn't. The legacy `sandbox` value is still required at app-server compatibility boundaries (the protocol shape `ThreadStartResponse.sandbox` and the rollout-thread `metadata.sandbox_policy`), so the projection is computed on demand via `codex_sandboxing::compatibility_sandbox_policy_for_permission_profile(profile, fs_policy, network_policy, cwd)`.

This is PR-3 in a 5-PR Sapling stack (#19772 → #19773 → #19774 → **#19775** → #19776) carving permission state down to a single canonical source.

## Verdict: `merge-after-nits`

The "store one, derive the other" direction is correct. The projection is pure (no I/O, no caching), call sites flip in lockstep with the field removal so the compiler enforces no orphan reads, and one pre-existing test was updated to call the projection. Two nits and one architectural follow-up.

## Specific references

- `codex-rs/core/src/codex_thread.rs:50` — `sandbox_policy: SandboxPolicy` field is *removed* from `ThreadConfigSnapshot`. Because `SandboxPolicy` is `pub` and the struct is `pub`, this is a public-API break for any downstream crate that constructed snapshots with named fields — but the destructure pattern at `codex_message_processor.rs:9001-9007` (`let ThreadConfigSnapshot { model, ..., permission_profile, cwd, ... } = pending.config_snapshot;`) drops the `sandbox_policy` binding cleanly, confirming all in-tree call sites are updated.
- `codex-rs/core/src/codex_thread.rs:58-67` — the new projection:
  ```rust
  impl ThreadConfigSnapshot {
      pub fn sandbox_policy(&self) -> SandboxPolicy {
          let file_system_sandbox_policy = self.permission_profile.file_system_sandbox_policy();
          codex_sandboxing::compatibility_sandbox_policy_for_permission_profile(
              &self.permission_profile,
              &file_system_sandbox_policy,
              self.permission_profile.network_sandbox_policy(),
              self.cwd.as_path(),
          )
      }
  }
  ```
  Three observations: (a) this is *recomputed* on every call, including inside hot paths like the thread-resume mismatch loop (`codex_message_processor.rs:9230` calls it once per `requested_sandbox` check, OK because there's only one); (b) the result is not memoized on `&self`, but `compatibility_sandbox_policy_for_permission_profile` should be deterministic and cheap (it's switching on enum variants and constructing a `SandboxPolicy::WorkspaceWrite { writable_roots: ... }` with the cwd-derived root); (c) `file_system_sandbox_policy()` is computed twice — once for the explicit arg, once internally if the upstream helper recomputes — worth a quick check to confirm the helper doesn't re-derive what's already passed in.
- `codex-rs/app-server/src/codex_message_processor.rs:2862,2875` — first migrated call site captures `let sandbox_policy = config_snapshot.sandbox_policy();` then uses `sandbox_policy.into()` for the protocol field. The named-binding-then-use pattern (vs. inlining `config_snapshot.sandbox_policy().into()`) is borrow-checker friendly: `into()` consumes the value, so the explicit binding makes it clear this is a *projection that produces a fresh owned value each call*, not a field access.
- `codex-rs/app-server/src/codex_message_processor.rs:3601` — `builder.sandbox_policy = config_snapshot.sandbox_policy();` (no `.clone()` needed anymore — the projection already returns owned). Removes a `.clone()` from the previous shape, small win.
- `codex-rs/app-server/src/codex_message_processor.rs:9229-9252` — `collect_resume_override_mismatches` is the most subtle migration: the comparison branch now binds `let active_sandbox = config_snapshot.sandbox_policy();` *before* the `matches!` so the `&active_sandbox` reference outlives the match, and the error-message format string at L9249 was updated to interpolate `{active_sandbox:?}` instead of `{:?}` of the field. Both are required because the projection returns owned, not borrowed.
- `codex-rs/core/src/memories/tests.rs:721,755` — pre-existing test `phase2::*` updated to bind `let sandbox_policy = config_snapshot.sandbox_policy();` once and use it for both the variant match and the `get_writable_roots_with_cwd` call. Captures the new "compute once, use many" idiom.

## Nits

1. The projection is named `sandbox_policy()` — the same identifier as the removed field. That makes call-site diffs read as `s.sandbox_policy` → `s.sandbox_policy()` (one-paren change), which is great for diff comprehension but also means the call site looks identical to a field access during code review of *future* PRs touching this surface. Consider renaming to `derived_sandbox_policy()` or `compatibility_sandbox_policy()` to make the projection nature legible at the call site. (Existing PR-stack norms may push back on this; mention it in the stack discussion not as a blocker.)
2. There's no `#[doc]` on the new method explaining *why* it's a projection, what its invariants are, or that `permission_profile` is the source of truth. Future contributors who see `sandbox_policy()` will be tempted to add a backing field for "performance". A two-line `///` doc comment would forestall that:
   ```rust
   /// Derives the legacy `SandboxPolicy` shape from the canonical
   /// `permission_profile` + `cwd`. Recomputed on each call by design;
   /// do not cache — `permission_profile` is the source of truth.
   ```
3. Snapshot test `phase2::*` was updated mechanically (field reads became method calls) but no *new* test was added to pin the invariant "two `ThreadConfigSnapshot`s with the same `permission_profile` and `cwd` produce equal `sandbox_policy()` results, regardless of when they were constructed." That's the regression door this PR is supposed to close — no new test means the door is closed structurally (the field can't drift because it doesn't exist) but a one-shot test would document the contract for future readers.

## Architectural follow-up

The 5-PR stack as a whole is collapsing several parallel "permissions" representations down to `PermissionProfile` as canonical. After this PR lands and #19776 ("store thread sessions as profiles") follows, the natural next move is auditing `codex-protocol` for any wire-format that still emits `sandbox` redundantly with `permission_profile` — over-the-wire duplication invites the same drift bug at the *client* side. Worth flagging in the stack's tracking issue.

## What I learned

"Store one, derive the other" is the right shape whenever two representations of the same state exist for compatibility reasons, but only if the projection is (a) cheap, (b) total (no failure path), (c) deterministic, and (d) name-disambiguated from a field access. This PR satisfies (a)-(c) and pays slightly on (d) by keeping the same identifier. The reason the pattern works *here* and not always is that the legacy `SandboxPolicy` and canonical `PermissionProfile` aren't bijective — `PermissionProfile` is strictly *more* expressive (it can represent split filesystem rules, disabled enforcement, and external enforcement that don't round-trip through `SandboxPolicy`) — so the only safe direction is canonical → legacy on demand. If a previous PR had picked the opposite direction, the projection would have been lossy and the drift bug would have moved into the projection itself instead of being eliminated.
