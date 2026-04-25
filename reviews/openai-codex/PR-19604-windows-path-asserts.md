# PR: test: stabilize app-server path assertions on Windows

- URL: https://github.com/openai/codex/pull/19604
- Author: bolinfest
- Head SHA: `8be36acf9d5415c5229b5b596962176f91f9dfc8`
- State: OPEN  (+1388 / −658, 63 files)

## Summary

The stated goal is to remove hardcoded `"/tmp"` / `r"C:\temp"` literals from app-server tests so assertions don't drift between platforms. The actual change is much larger: it introduces a workspace-wide use of `AbsolutePathBuf::from_absolute_path(std::fs::canonicalize(...))` in tests, adds a Windows-aware `contains_glob_chars_for_platform` and `normalize_windows_device_path` helper to `utils/absolute-path`, and refactors several production code paths (`sandbox_tag` → `permission_profile_sandbox_tag`, `command_exec.rs`, exec/sandboxing modules) under the same PR.

## Specific observations

- `codex-rs/Cargo.lock:2870` — removes a dependency on `codex-utils-absolute-path` from one crate while another file (`codex-rs/utils/absolute-path/src/lib.rs`) is being expanded. The lockfile delta should be a side-effect of explicit `Cargo.toml` edits; verify each manifest change is intentional and not an accidental drop that will surface on the next `cargo update`.
- `codex-rs/utils/absolute-path/src/lib.rs` (new helper `normalize_path_for_platform`) — the `if cfg!(windows) && let Some(path) = path.to_str() && let Some(normalized) = normalize_windows_device_path(path)` chain uses Rust 2024-style `let` chains. Confirm MSRV in the workspace's root `Cargo.toml` permits this; if MSRV is 1.78 or earlier, the build breaks on the pinned toolchain. (Codex generally tracks recent stable, but this is worth confirming on a release-engineering-flavored PR.)
- `codex-rs/core/src/sandboxing/mod.rs` (`sandbox_tag` → `permission_profile_sandbox_tag`) — this is a rename in production code, not a test stabilization. The diff at the rename site changes the *value* used to construct the `sandbox` field on a session metadata record (`Some(permission_profile_sandbox_tag(permission_profile, windows_sandbox_level).to_string())`). That is observable to downstream telemetry consumers and to anyone parsing rollout JSON. It does not belong in a "stabilize Windows test assertions" PR.
- `codex-rs/exec-server/src/fs_sandbox.rs`, `codex-rs/exec/tests/suite/sandbox.rs`, `codex-rs/linux-sandbox/tests/suite/landlock.rs`, `codex-rs/core/src/exec.rs`, `codex-rs/core/src/exec_tests.rs` — multi-file changes to exec and sandboxing modules across Linux, exec-server, and the cross-platform exec layer. None of these are the `app-server` path-assertion stabilization the title promises.
- `codex-rs/core/src/context/permissions_instructions.rs` and `…_tests.rs` — `assert!(text.contains("Network access is enabled."))` plus `assert!(text.contains(writable_root.to_string_lossy().as_ref()))`. Switching from a hardcoded literal to `to_string_lossy().as_ref()` is the canonical Windows-portable assertion; that's the actual stated work and it's done correctly.
- `codex-rs/app-server/src/codex_message_processor.rs` (`@@ -2272 +2273` etc.) — substantial restructuring of message-processor methods, not a test fix. Production file in an app-server PR with title "test:".
- 63 files / +1388 / −658 for a "test stabilization" PR is far outside what the title promises. A reviewer asked to ack a Windows-test stabilization will not have the right context for the production rename and the sandboxing refactors.

## Verdict

`request-changes`

## Reasoning

The portable-test-assertion mechanics (replacing `r"C:\temp"` literals with `AbsolutePathBuf::from_absolute_path(std::fs::canonicalize(...))` and `to_string_lossy()`) are correct and welcome. But the PR also renames a production helper (`sandbox_tag` → `permission_profile_sandbox_tag`) in a way that mutates user-visible session metadata, restructures `codex_message_processor.rs`, and adds a `let`-chain helper to a shared utility crate. Each of those needs its own focused review and changelog entry. Split the test-portability work from the production refactor and the rename, land the test-only piece quickly (it is genuinely unblocking for Windows CI), and re-PR the rest with descriptive titles so reviewers can give them the attention they deserve.
