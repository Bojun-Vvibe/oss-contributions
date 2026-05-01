# openai/codex #20658 — [codex-exec] Preserve add-dir roots for macOS exec sandbox

- URL: https://github.com/openai/codex/pull/20658
- Head SHA: `628b295899f87dc3a43dfe40d299354941652646`
- Files: `codex-rs/exec/src/lib.rs` (+84/-4), `codex-rs/exec/src/lib_tests.rs` (+61/-0)
- Tracks BUGB-15632

## Context / problem

`codex exec` on macOS routes through the in-process app server, and when it projected the `Config` into thread-start / thread-resume params it dropped the `--add-dir` writable roots that the user passed on the command line. The direct Seatbelt backend honored them; the app-server path silently lost them. Net effect: a user said `codex exec --add-dir /workspace/backend ...`, watched the agent succeed at the direct-Seatbelt level, then watched the same command fail under `just c exec` because the writable root never made it across the param boundary.

## Design analysis

The fix at `exec/src/lib.rs:1018-1100` rebuilds the projection logic. Previously `config_request_overrides_from_config` returned only the active named profile name (one HashMap entry). Now it composes:

1. Active profile name (unchanged path).
2. **New** `add_legacy_workspace_write_overrides` helper that, **only** when no named profile is active (`config.permissions.active_permission_profile().is_some() => return`), reads the runtime-projected `(file_system_policy, network_policy)` and rebuilds a `sandbox_workspace_write` JSON blob from the canonical `FileSystemSandboxPolicy` entries — the same source of truth the direct-Seatbelt backend already used.

The composition discipline is correct:

- `has_writable_project_roots_entry` at `:1083-1090` gates the whole rehydration on the policy actually being a "workspace-write" shape (writable `ProjectRoots { subpath: None }`); other policy shapes pass through to the existing path. This avoids accidentally enabling workspace-write on a policy that didn't ask for it.
- `explicit_writable_roots` at `:1063-1078` walks `file_system_policy.entries`, filters to `FileSystemPath::Path { path }` with `entry.access.can_write()`, deduplicates by linear scan (fine for the cardinality — `--add-dir` is rarely more than a handful of paths), and returns a `Vec<AbsolutePathBuf>`.
- `network_access` and the two `exclude_*` booleans at `:1037-1043` are derived from the same canonical policy so a user who passed `--add-dir foo` while also opting into network access or `/tmp` writability gets all four signals preserved across the param boundary, not just the writable root.
- The `(!overrides.is_empty()).then_some(overrides)` at `:1027` preserves the original "no overrides → return None" contract, so callers that don't care about active profile or workspace-write rehydration still see `None` rather than an empty map.

The regression test at `lib_tests.rs:425-481` is dispositive:
- Builds a `ConfigBuilder` with `cwd: Some(frontend)` and `additional_writable_roots: vec![backend.clone()]` (the canonical `--add-dir` shape).
- Computes both `thread_start_params_from_config` AND `thread_resume_params_from_config` (catches the resume-path dropping the root, which is how the original bug would re-emerge after a session restart).
- Asserts `sandbox_workspace_write.writable_roots` contains the absolute path of `backend` for both.
- Also asserts `sandbox` is `WorkspaceWrite` and `permissions` is `None` to lock the legacy-vs-named-profile partitioning.

## Risks

- The `has_writable_project_roots_entry` gate's `subpath.is_none()` check is the load-bearing predicate that decides "is this the legacy workspace-write shape?" If a future contributor adds workspace-write profiles that use `subpath: Some(...)`, those will silently fall through to the empty-overrides path and lose `--add-dir` again. Worth a comment at `:1083-1090` flagging that this predicate must be kept in sync with the canonical workspace-write definition.
- The dedup via linear scan at `:1072-1074` is `O(n²)` — fine for `n ≤ ~10` but if anyone ever passes `--add-dir` 100+ times this is wasted work. Not blocking.
- The PR honestly notes a pre-existing failing test (`thread_start_params_include_review_policy_when_review_policy_is_manual_only`) reproducible on clean main at `6784db51c07c3d35e06685024fce0cd24c419f34` — that's not this PR's regression.

## Suggestions

- Add the `subpath.is_none()` "keep in sync with workspace-write definition" comment described above.
- Consider extracting `add_legacy_workspace_write_overrides` to a `cfg(test)`-callable helper that asserts the projection round-trips against the direct-Seatbelt path's view of the same `Config` — would catch any future drift between the two consumers of `FileSystemSandboxPolicy`.

## Verdict

`merge-after-nits` — correct fix for a real "two backends disagree on the projection of one config" bug, with the rehydration sourced from the canonical `FileSystemSandboxPolicy` rather than re-deriving from the legacy `SandboxPolicy` projection (which would have re-introduced the same drift in a different shape), gated on the legacy workspace-write shape so it doesn't perturb named-profile callers, regression-tested across both start and resume params with the dispositive `additional_writable_roots` shape, and validated against actual macOS Seatbelt E2E (`pass=8 fail=0`). Wants the keep-in-sync comment + the round-trip assertion helper.
