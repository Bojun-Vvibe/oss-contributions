# openai/codex PR #19435 — Allow manual unified_exec opt-in on Windows

- **Repo:** openai/codex
- **PR:** [#19435](https://github.com/openai/codex/pull/19435)
- **Head SHA:** `f4976eeac19b4ae46978b0bb5588d4c02d2fc724`
- **Author:** iceweasel-oai
- **Size:** +28/-71 across 3 files
- **Reviewer:** Bojun (drip-24)

## Summary

Subtractive PR (more deleted than added). Removes the
Windows-specific environment gate
(`unified_exec_allowed_in_environment`) that prevented Windows
sandboxed sessions from ever selecting `unified_exec`, and
replaces the model-provided shell-type fallback with a simpler
"if feature flag off, normalize to `ShellCommand`" check.

Net effect: Windows users can now opt into `unified_exec`
manually via the feature flag, while still getting `ShellCommand`
by default. macOS / Linux behavior is unchanged because the old
gate only bit on Windows.

## Key changes

### `tools/src/tool_config.rs` (+8/-25)

The destructure pattern at line 134 changes from:

```rust
let ToolsConfigParams { ..., sandbox_policy, windows_sandbox_level } = params;
```

to `..` — both fields are no longer used in this function. That's
the tell that the environment gate is gone.

The new shell_type resolution (lines 168–186):

```rust
let unified_exec_enabled = features.enabled(Feature::UnifiedExec);
let model_shell_type = match model_info.shell_type {
    ConfigShellToolType::UnifiedExec if !unified_exec_enabled => {
        ConfigShellToolType::ShellCommand
    }
    other => other,
};
let shell_type = if !features.enabled(Feature::ShellTool) {
    ConfigShellToolType::Disabled
} else if features.enabled(Feature::ShellZshFork) {
    ConfigShellToolType::ShellCommand
} else if unified_exec_enabled {
    if codex_utils_pty::conpty_supported() {
        ConfigShellToolType::UnifiedExec
    } else {
        ConfigShellToolType::ShellCommand
    }
} else {
    model_shell_type
};
```

Two semantic shifts:

1. **Model-provided `unified_exec` is now feature-gated,
   not env-gated.** Old code: model says `UnifiedExec`, server
   checks `unified_exec_allowed_in_environment(...)`, falls back
   to `ShellCommand` on Windows-sandboxed. New code: model says
   `UnifiedExec`, server checks `Feature::UnifiedExec`, falls back
   to `ShellCommand` if the user hasn't opted in. **This is
   stricter** — even on macOS/Linux where the old gate would have
   said "yes, allowed", a user without the feature flag now gets
   `ShellCommand`.
2. **The `unified_exec_allowed && conpty_supported()` Windows
   ceremony is preserved** for the user-opted-in path (line 178)
   — if the user enables `Feature::UnifiedExec` on a Windows
   machine without ConPTY support, they still get `ShellCommand`,
   not a broken UnifiedExec. Good.

### `tools/src/tool_config_tests.rs` (+15/-22)

The old test
`unified_exec_is_blocked_for_windows_sandboxed_policies_only`
which exercised the four-arm permission table is gone. Replaced
with `model_provided_unified_exec_requires_feature_flag` which
asserts: feature off + model says `UnifiedExec` + DangerFullAccess
+ Windows sandbox disabled → `ShellCommand`.

This test only covers **one** of the matrix cells. The old test
covered four. Specifically, the new test does NOT cover:

- feature off + model says `UnifiedExec` + workspace-write policy
  (Windows-sandboxed)
- feature on + Windows-sandboxed (does the new code correctly
  give UnifiedExec? It should — the old gate is gone)
- feature on + Windows + ConPTY unsupported (should fall back to
  ShellCommand)

### `core/src/tools/spec_tests.rs` (-25)

Drops `model_provided_unified_exec_is_blocked_for_windows_sandboxed_policies`
entirely. This is the test whose contract is now intentionally
broken by the PR — it asserted Windows-sandboxed + workspace-
write → `ShellCommand` regardless of feature flag. The new
behavior is "if feature on, give UnifiedExec". So deletion is
correct, but I'd want a *replacement* test that pins the new
behavior: feature on + Windows-sandboxed → UnifiedExec.

## Concerns

1. **Test matrix shrinkage.**

   Old: 4 explicit assertions in
   `unified_exec_is_blocked_for_windows_sandboxed_policies_only`
   covering (Windows + RestrictedToken + read-only / workspace-
   write / DangerFullAccess) and (Windows + Disabled +
   DangerFullAccess). New: 1 test covering the
   "model-provided + feature-off → ShellCommand" arm. The deleted
   test in `spec_tests.rs` (which exercised the 'model says
   UnifiedExec, server normalizes to ShellCommand on Windows' case
   under the old env gate) is gone with no replacement asserting
   the new behavior. **At minimum, add a test asserting that
   feature-on + Windows-sandboxed-workspace-write yields
   `UnifiedExec` (or `ShellCommand` if ConPTY unsupported).**

2. **Default behavior on Windows is now controlled by feature
   flag default.**

   `Features::with_defaults().enabled(Feature::UnifiedExec)` —
   need to confirm this is `false` on Windows. The PR description
   says "keep `unified_exec` default-off on Windows unless the
   feature is explicitly enabled", but the diff doesn't show the
   default value. If `Feature::UnifiedExec` defaults to `true`
   anywhere (e.g. via OS-conditional compile), this PR silently
   opens up unified_exec for Windows sandboxed sessions. Worth
   verifying in `features.rs` (not in the diff).

3. **Behavior change for non-Windows users who hadn't opted in.**

   Pre-PR: model says `UnifiedExec`, non-Windows user, feature
   off → `unified_exec_allowed_in_environment(false, ...)` returns
   `true` (the function only blocks on Windows-sandboxed), so the
   final fallback `model_info.shell_type` returns `UnifiedExec`.

   Post-PR: model says `UnifiedExec`, non-Windows user, feature
   off → `model_shell_type` is normalized to `ShellCommand`.

   So **macOS/Linux users without `Feature::UnifiedExec` who were
   previously getting `UnifiedExec` from a model declaration will
   now get `ShellCommand`**. The PR description claims "macOS and
   Linux behavior is unchanged" — that's only true if no model
   was previously declaring `UnifiedExec` *and* the user had
   the feature off. If any deployed model already declares this,
   this is a silent regression for non-opted-in users.

## Verdict

`request-changes` — the structural cleanup is good (env gate
removal is overdue, the destructure-with-`..` is right), but the
test coverage shrinkage and the silent macOS/Linux behavior
change combine to make this riskier than it looks. Specifically:

- add a regression test for feature-on + Windows-sandboxed
  yielding `UnifiedExec` (the new positive path),
- add a regression test for the macOS/Linux + feature-off +
  model-says-UnifiedExec case, asserting whichever behavior is
  intended,
- update the PR description to either (a) explicitly call out
  the macOS/Linux silent normalization, or (b) preserve the
  pre-PR behavior on non-Windows by checking
  `cfg!(target_os = "windows")` in the `model_shell_type`
  normalization arm.

If the answer is "yes, we want to normalize on all platforms",
that's defensible — but it should be a documented contract
change, not an accidental side effect of the Windows refactor.

## What I learned

Subtractive PRs that delete a permission gate have to be read
twice: once for "what does the new code do" and once for "what
matrix cells did the old code cover that the new code doesn't".
The deleted `unified_exec_allowed_in_environment` function had a
non-obvious second responsibility — it was also the gate for
"model declares UnifiedExec but the env doesn't support it" on
non-Windows, even though the function name suggested
Windows-only. That dual responsibility is the source of the
silent macOS/Linux normalization. The lesson: when you delete
a helper, grep for *every* callsite + every conditional that
depends on its return value, not just the obvious "is X
allowed" check.
