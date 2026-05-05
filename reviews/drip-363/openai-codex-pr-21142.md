# openai/codex #21142 — Support multi-env shell_command routing

- Head SHA: `e9a56cb2c5249e94b8b0d090e528ee9e5eb68034`
- Author: starr-openai
- Diff: +549 / −40 across 7 files
- Stack: PR-2 of a series; depends on #20647 (which added selected-environment routing for `exec_command`).

## Verdict: `merge-after-nits`

## What it does

Brings `shell_command` to the same multi-environment contract that `exec_command` and `view_image` already use:

1. **Optional `environment_id` arg.** Surfaced when multiple envs are selected; falls back to the primary env otherwise (`shell.rs:431-440`).
2. **`workdir` resolved against the selected env's cwd, not the turn cwd.** A new `ShellCommandEnvironmentArgs` is parsed *before* the full `ShellCommandToolCallParams`, so the base path used to resolve relative path-like arguments is `turn_environment.cwd.join(workdir)` instead of `turn.cwd` (`shell.rs:444-450`). This is the main correctness change — without it, a remote env could see the model's `workdir: "src"` resolved against the controller's local cwd.
3. **Remote exec-server routing.** When `turn_environment.environment.is_remote()` returns true, an `ExecServerEnvConfig` is built from the active `ShellEnvironmentPolicy` and wired through `RunExecLikeArgs` so the request can be dispatched to the remote exec-server with the right env policy (`shell.rs:467-480`, `exec_server_env_config()` at `shell.rs:208-217`).
4. **Sandbox + approval cwd uses the selected env's cwd.** `sandbox_cwd: exec_params.cwd.as_path()` instead of `turn.cwd.as_path()` at `shell.rs:614` and `apply_granted_turn_permissions(... exec_params.cwd.as_path() ...)` at `shell.rs:528` — symmetric with the workdir-resolution change so the sandbox profile actually matches the directory the command will run in.
5. **Per-environment shell.** `get_shell_by_model_provided_path(&PathBuf::from(&turn_environment.shell))` (`shell.rs:462`) replaces the previous `session.user_shell()` so a remote env's configured shell wins over the controller's session shell.

## What's good

- **Clean separation of envelope vs. payload parsing.** `ShellCommandEnvironmentArgs` is its own `serde::Deserialize` struct (`shell.rs:99-107`) with a comment explicitly explaining why `workdir` is parsed raw rather than as `AbsolutePathBuf` — relative paths must wait until env selection. That comment is exactly the right level of "why" for a future reader who's tempted to fold the structs together.
- **Test rename matches the semantic shift.** `shell_command_handler_to_exec_params_uses_session_shell_and_turn_context` → `..._uses_selected_shell_and_turn_context` (`shell_tests.rs:79`) and the test body now passes the resolved `shell` + `cwd` directly, mirroring the new `to_exec_params` signature.
- **`exec_server_env_policy_from_shell_policy` mapper is total** — every field of `ShellEnvironmentPolicy` is forwarded with no defaulting (`shell.rs:189-205`), which is the right call for an env-policy mapper (silent defaults here would be a security footgun).
- **Backward-compatible for the non-multi-env caller.** `ShellHandler` (the legacy `shell` tool) gets `target_environment` populated from `turn.environments.primary()` with `remote_execution_enabled: false` and `exec_server_env_config: None` (`shell.rs:300-310, 329-339`), so existing single-env behavior is preserved.

## Nits worth raising before merge

1. **`environment_id` validation error is opaque.** When `resolve_tool_environment` returns `None` at `shell.rs:438`, the error is `"shell is unavailable in this session"` — same string as the no-primary-env case. A model that passes a *bad* `environment_id` (typo, stale id from a prior turn) gets the same message as the truly-unavailable case and has no way to recover. Consider differentiating: `"environment_id '<id>' not found; available: <ids>"` for the invalid-id case.

2. **`workdir.is_empty()` check is string-empty, not whitespace-empty.** At `shell.rs:445-449`:
   ```rust
   environment_args.workdir.as_deref()
       .filter(|workdir| !workdir.is_empty())
       .map_or_else(|| turn_environment.cwd.clone(), ...)
   ```
   A `workdir: " "` (single space) would pass through to `turn_environment.cwd.join(" ")`, which is unlikely to do anything useful and is harder to debug than just treating whitespace-only as absent. `workdir.trim().is_empty()` is one extra char and removes a class of "why is my command running in the wrong dir" bug reports.

3. **`exec_server_env_config` is built from `exec_params.env` *after* `create_env(...)` has already merged the policy** — but `local_policy_env: exec_params.env.clone()` (`shell.rs:218-221`) then ships *that already-merged* env to the remote exec-server alongside a *separate* `policy` derived from the same source. If the remote re-applies the policy on top of `local_policy_env`, the policy effectively runs twice. Worth a comment confirming this is intentional (e.g. "remote re-applies policy as a defense-in-depth check") or a refactor that ships only the unmerged inputs.

4. **`exec_params.cwd.as_path()` is now used in three places** — `apply_granted_turn_permissions` (`shell.rs:528`), `sandbox_cwd` (`shell.rs:614`), and the `ShellRequest` itself (`shell.rs:628`). All three previously used `turn.cwd`. The asymmetry with the legacy `ShellHandler` path (which still has `target_environment` derived from `turn.environments.primary()` so its `exec_params.cwd` may not have changed) is fine for now, but if `ShellHandler` ever gains its own `environment_id` plumbing, the `apply_granted_turn_permissions` cwd argument will need to be revisited per-call rather than relying on the new symmetry.

5. **No test for the `is_remote()` branch.** The diff exercises the renamed test and the new-shell-derivation path, but `exec_server_env_config` being `Some(...)` only when `is_remote()` is true has no covering assertion. A two-line test that constructs a remote `Environment` and asserts `args.exec_server_env_config.is_some()` would pin the contract.

## Why not request-changes

The five points above are all surface-level: validation messages, whitespace handling, a doc comment, and a missing test. The core behavior change (per-env workdir resolution, per-env shell, per-env sandbox cwd, remote exec-server routing) is internally consistent and the legacy single-env caller is preserved. The author's own validation note ("Local Cargo validation is blocked by the Codex repo guardrail that requires devbox/Bazel validation") is a known repo constraint, not a PR defect.

## Citations

- `codex-rs/core/src/tools/handlers/shell.rs:99-107` — `ShellCommandEnvironmentArgs` struct
- `codex-rs/core/src/tools/handlers/shell.rs:189-221` — env-policy mapper + `exec_server_env_config` builder
- `codex-rs/core/src/tools/handlers/shell.rs:431-480` — env resolution, workdir resolution, exec-server config build
- `codex-rs/core/src/tools/handlers/shell.rs:528, 614, 628` — three sites switched from `turn.cwd` to `exec_params.cwd`
- `codex-rs/core/src/tools/handlers/shell_tests.rs:79-115` — rename + new signature test
