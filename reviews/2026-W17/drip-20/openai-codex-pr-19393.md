---
pr_number: 19393
repo: openai/codex
head_sha: 07b8227db72c2ac9dfdbac3a67815f3ecbb647d9
verdict: merge-after-nits
date: 2026-04-25
---

# openai/codex#19393 — migrate approval/sandbox consumers to `PermissionProfile`

**What changed.** +386 / −216. Five surfaces flip from `SandboxPolicy` to `PermissionProfile` as the source of truth: (a) `analytics/src/facts.rs:64` — `TurnResolvedConfigFact` swaps `sandbox_policy: SandboxPolicy` for `permission_profile: PermissionProfile` + `permission_profile_cwd: PathBuf`; (b) `analytics/src/reducer.rs:887` consumes the new fields and `sandbox_policy_mode` (line 962) is rewritten to match `PermissionProfile::Disabled` → `"full_access"` and `PermissionProfile::External` → `"external_sandbox"` directly, falling back to `to_legacy_sandbox_policy(cwd)` only for `Managed`; (c) `app-server/src/codex_message_processor.rs:2151` — `start_proxy` now receives `permission_profile.get()` instead of `sandbox_policy.get()`; (d) the same file, lines 2215–2275, threads runtime perms via `permission_profile.to_runtime_permissions()` and re-validates with `permission_profile.can_set(&effective_permission_profile)`; (e) the legacy fallback path at line 2266 still exists but now also passes through the same `can_set` validator on the `PermissionProfile` form.

**Why it matters.** Per the PR body: previously `Disabled` and `External` both projected to lax-ish `SandboxPolicy` shapes that approval/network/Landlock checks could not distinguish. The classic example: `External` should still let an outer sandbox impose network restrictions; `Disabled` is "no sandbox, do anything." With `SandboxPolicy` as the gate, both routed identically. Same problem applied to deny-read entries on split filesystem profiles getting flattened by the projection.

**Concerns.**
1. **`sandbox_policy_mode` `Err(_)` arm at line 970** silently labels failures as `"workspace_write"` for analytics. That's not zero-information — it's *wrong* information attributed to whatever default we pick. At minimum, increment a `permission_profile_to_legacy_failed` counter and log at WARN once per process; analytics buckets being silently miscategorized is exactly the kind of bug that surfaces in a quarterly cohort review six weeks late.
2. **`preserve_configured_deny_read_restrictions` mutation order.** Line 2222 calls `to_runtime_permissions()`, then line 2226 mutates the returned `file_system_sandbox_policy`, then line 2231 reconstructs the profile via `from_runtime_permissions_with_enforcement`. This is a non-atomic round-trip: enforcement mode is preserved, but anything else `to_runtime_permissions()` decided based on profile-internal state (e.g. `glob_scan_max_depth`, project-roots metadata) survives only if `from_runtime_permissions_with_enforcement` reads it back from the *file_system_sandbox_policy* arg. Confirm with a regression test that a `Managed` profile with `glob_scan_max_depth = Some(2)` survives `request_runtime_permission_change` end-to-end.
3. **Legacy fallback at line 2253–2273** still goes through the materialized `from_legacy_sandbox_policy_for_cwd` AND then `from_runtime_permissions_with_enforcement` AND then `can_set` again. Three transforms for the legacy path versus one for the new path. The legacy code is presumably scheduled to die in a follow-up — leave a `// TODO(#xxxxx): drop after legacy SandboxPolicy callers are gone` so it doesn't ossify.
4. **Reducer test (`analytics_client_tests.rs:315`)** wires `permission_profile_cwd: PathBuf::from("/tmp")` directly. On Windows this is *not* an absolute path. If the test runs on Windows, `to_legacy_sandbox_policy` will fail and the fallback `"workspace_write"` arm fires — silently passing the test for the wrong reason. Use `std::env::temp_dir()` or `if cfg!(windows) { PathBuf::from(r"C:\Temp") }`.
5. Stack PR. Don't land standalone; merge with #19391/#19392/#19394/#19395/#19414 in stack order.

Net-positive direction (Disabled vs External finally distinguishable). Land after the silent-`Err` log and the deny-read round-trip test.
