---
pr_number: 19414
repo: openai/codex
head_sha: c86d16a254199c3e56d28942613c77e3911f2c43
verdict: merge-after-nits
date: 2026-04-25
---

# openai/codex#19414 — make legacy `PermissionProfile` conversion cwd-free

**What changed.** 29 files / +254 / −137. The single semantic change is in `codex-rs/protocol/src/permissions.rs` (+180/−41): `FileSystemSandboxPolicy::from_legacy_sandbox_policy` is now the symbolic, cwd-free projection (workspace-write becomes `:cwd = write` + read-only `:project_roots` special entries), and the old absolute-path-materializing conversion is renamed `from_legacy_sandbox_policy_for_cwd`. `PermissionProfile::from_legacy_sandbox_policy` (called from `analytics_client_tests.rs:163`, `codex_message_processor.rs:10464`, `session/tests.rs:1497`, `tui/src/app/tests.rs`, etc.) drops its `cwd` parameter entirely. Every runtime/landlock/seatbelt/config call site that genuinely needs cwd materialization is renamed in lockstep — `core/src/landlock.rs:39`, `core/src/config/mod.rs:1866`, `core/src/memories/phase2.rs:324`, `core/src/safety_tests.rs:181`, `core/src/safety_tests.rs:303`, `app-server/src/codex_message_processor.rs:2267`, `core/src/session/session.rs:124`, `core/src/session/session.rs:204` — all now call `_for_cwd`.

**Why it matters.** Profile producers were inventing an ambient cwd just to satisfy a parameter that the symbolic projection didn't actually need. That ambient cwd was a footgun: it baked an absolute path into something destined to be re-applied against a *different* cwd later (e.g. on resume, or when a sub-agent inherits a parent profile). Stack PR — sits on top of #19391/#19392/#19393/#19394 and below #19395.

**Concerns.**
1. **Naming asymmetry.** `from_legacy_sandbox_policy` now returns a *symbolic* policy, but `from_legacy_sandbox_policy_for_cwd` returns a *materialized* policy. Casual readers will assume the difference is "with vs without explicit cwd", not "abstract vs concrete". Consider `from_legacy_sandbox_policy_symbolic` + `from_legacy_sandbox_policy_concrete` (or add a 2-line doc-comment block on each that names the projection).
2. **`codex_message_processor.rs` mixes both APIs in the same function.** Line 2267 uses `_for_cwd` (correct, runtime path), but the test fixtures starting at line 10464 use the cwd-free variant (also correct, test path). Easy to slip during future edits — would be safer if the tests acquired a `_for_cwd` form via a helper that takes `test_path_buf("/tmp")` so a future mistake routing the cwd-free variant into a runtime call site is structurally rejected.
3. **`landlock.rs:39` and `seatbelt.rs:~50`** unconditionally use `_for_cwd`. Good — that's what the underlying syscalls need. But verify on Linux that `from_legacy_sandbox_policy_for_cwd(&SandboxPolicy::DangerFullAccess, …)` still produces an unrestricted policy; the cwd-free variant for `DangerFullAccess` is trivially unrestricted, but the materialized one historically went through `WorkspaceWrite { cwd, … }` for one branch. The diff doesn't show the `DangerFullAccess` arm of `from_legacy_sandbox_policy_for_cwd`, so this is on the reviewer to confirm in the file.
4. **`protocol/src/permissions.rs` +180 lines** is the meat. Without the full file in the diff window I can see only the renames; new tests for the `:cwd = write` + `:project_roots = read` symbolic representation are presumably part of those 180 lines. Confirm there's a round-trip test: `WorkspaceWrite{cwd, writable_roots, exclude_slash_tmp}` → symbolic profile → `to_legacy_sandbox_policy(cwd)` → original `WorkspaceWrite`.
5. **Stack ordering.** Body says "Stack" with #19414 at the top of the list. `Sapling` stacks usually order children-first; if #19414 actually depends on the four below it, do not land standalone.

Mechanical, well-scoped rename. Land after a docstring pass and the `DangerFullAccess` parity check.
