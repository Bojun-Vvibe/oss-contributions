# PR #19841 — permissions: remove cwd special path

- **Repo**: openai/codex
- **PR**: #19841
- **Head SHA**: `015d92761749a39eab5f12a11c763dbacb04e8e8`
- **Author**: bolinfest (Michael Bolin)
- **Size**: +147 / -426 across 46 files
- **Verdict**: **merge-after-nits**

## Summary

Pre-stabilization removal of the `FileSystemSpecialPath::CurrentWorkingDirectory`
variant from the experimental `PermissionProfile` API. Rationale
in the PR body (and supported by the diff) is sound: having both
`:cwd` and `:project_roots` as symbolic permission anchors made
the trusted-root semantics ambiguous, and a turn-cwd change
should affect command execution context — *not* silently rebind
durable filesystem authority. Callers that need symbolic
project-root scoping go through `:project_roots`; everything else
gets resolved to a concrete path before enforcement.

## Specific changes

- `codex-rs/protocol/src/permissions.rs` (+39/-39) drops the
  `CurrentWorkingDirectory` enum variant from
  `FileSystemSpecialPath` and renames the helper that produced
  the bundled-default profile to `legacy_workspace_write_template()`
  to flag that the result is a *symbolic* template requiring
  resolution before enforcement. The rename is doing the right
  semantic work — it forces every call site to acknowledge it's
  holding an unresolved template.
- `codex-rs/core/src/session/session.rs` (+3/-37) removes the
  cwd-only session-update branch that used to rebind permissions
  when cwd shifted; the new behavior anchors existing symbolic
  project-root permissions on cwd-only updates by materializing
  them against the *previous* permission root (referenced in the
  PR body, implementation in `session.rs`). Net effect: a cwd
  bump no longer secretly widens or narrows the permission set.
- `codex-rs/app-server-protocol/schema/json/*` — 18 schema files
  drop the `CurrentWorkingDirectoryFileSystemSpecialPath`
  oneOf-branch (~15 lines × 18 ≈ 270 deletions). Each diff is
  mechanical and safe; the JSON schema regeneration is what
  makes this PR legitimately big despite a small protocol change.
- `codex-rs/sandboxing/src/policy_transforms.rs` and
  `policy_transforms_tests.rs` — drop the cwd resolver branch
  and update 8 transform tests; the post-PR matrix is just
  `:project_roots` → concrete-paths.
- `codex-rs/exec-server/src/file_system.rs`, `fs_sandbox.rs`,
  `remote_file_system.rs` — strip the cwd-arm from the
  enforcement-path matches; nothing remains that would let a
  caller authoritatively pin to "wherever the turn is right
  now."
- Test additions in `core/src/exec_tests.rs` (+18/-6) and
  `core/src/session/tests.rs` (+28/-12) cover the cwd-update
  preserves-resolved-root case.

## Risks

1. **Wire-format break for any external consumer holding
   `kind: "current_working_directory"` payloads.** The PR title
   accurately says "remove" — there's no backwards-compat
   acceptance path. PR body explicitly says this is fine because
   the API isn't stabilized, which is the right call, but the
   release notes need to call out that any third-party UI or
   automation that constructed permission profiles with
   `:cwd`-style entries must migrate to `:project_roots` before
   pulling this in. Worth explicit `BREAKING:` prefix on the
   eventual changelog.
2. **`.json` schema files have no trailing newline.** Several
   regenerated schemas show `\ No newline at end of file` in the
   diff (e.g. `ClientRequest.json`, `codex_app_server_protocol.schemas.json`).
   That's a regeneration-tool quirk, not a correctness issue, but
   it'll cause noisy diffs on the next regen unless the generator
   is fixed to emit a trailing newline. Worth a follow-up to
   normalize.
3. **`legacy_workspace_write_template()` rename is a soft signal.**
   The "legacy" naming nudges callers toward "this needs to be
   resolved" but doesn't enforce it at the type level. A typestate
   wrapper (`UnresolvedTemplate<PermissionProfile>` →
   `.resolve(roots)` → `PermissionProfile`) would make it
   impossible to feed the template directly into enforcement.
   That's a follow-up refactor — out of scope for this PR.

## Verdict

`merge-after-nits` — the removal is correct, the test coverage
shifts in the right direction, and the rename surfaces the
previously-implicit "this is symbolic, resolve before use"
contract. Nits to land before merge: trailing-newline fix on the
regenerated JSON schemas, and a `BREAKING:` callout in the PR
title or release notes.

## What I learned

The "two ways to spell the same thing" footgun in permission
APIs (`:cwd` vs `:project_roots`) is the canonical case for
collapsing aliases *before* shipping a stable surface. Once
external code starts relying on either spelling, the cost of
removing one balloons; doing this in the experimental window is
the only sane time. The 270 lines of mechanical schema-file
deletions are the price of the protocol-as-source-of-truth
discipline — worth it because the JSON schemas *are* the
contract, and now they're internally consistent.
