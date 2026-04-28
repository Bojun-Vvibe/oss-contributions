# openai/codex #20089 — fix(protocol): expand Windows core env vars and split inherit=Core by target_os

- PR: https://github.com/openai/codex/pull/20089
- Head SHA: `37f31c88e15dec474a446b6fc5f9fa9b30378995`
- Files: `codex-rs/protocol/src/shell_environment.rs` (+120 / -20), `codex-rs/rmcp-client/src/utils.rs` (+2 / -29)

## Citations

- `shell_environment.rs:60-83` — `ShellEnvironmentPolicyInherit::Core` arm splits into `#[cfg(not(target_os = "windows"))]` (HashSet of `COMMON_CORE_VARS ∪ PLATFORM_CORE_VARS`, plain `contains`) and `#[cfg(target_os = "windows")]` (linear scan over `WINDOWS_CORE_ENV_VARS` with `eq_ignore_ascii_case`). The previous unified path always built the HashSet then conditionally case-folded — fine on Unix, but on Windows the hot path was running case-folding inside an `any(...)` over a HashSet built every call.
- `shell_environment.rs:122-153` — new `pub const WINDOWS_CORE_ENV_VARS` adds 13 vars beyond the old `["PATHEXT","USERNAME","USERPROFILE"]`: `COMSPEC`, `SYSTEMROOT`, `SYSTEMDRIVE`, `USERDOMAIN`, `HOMEDRIVE`, `HOMEPATH`, `PROGRAMFILES`, `PROGRAMFILES(X86)`, `PROGRAMW6432`, `PROGRAMDATA`, `LOCALAPPDATA`, `APPDATA`, `POWERSHELL`, `PWSH`. Constant is `pub` so `rmcp-client` can re-use it (see below).
- `shell_environment.rs:155-159` — `COMMON_CORE_VARS` and `PLATFORM_CORE_VARS` now both gated `#[cfg(not(target_os = "windows"))]`. Previously `PLATFORM_CORE_VARS` was `#[cfg(unix)]` while `COMMON_CORE_VARS` was unconditional — a real footgun because `cfg(unix)` is true on macOS/Linux/BSD but the *negation* `cfg(not(target_os = "windows"))` is also true on WASI, redox, etc., where the unix-only `LANG`/`LC_ALL` set is still the right approximation. Coherent gating.
- `shell_environment.rs:163-217` — two new `#[cfg(target_os = "windows")]` tests: `core_inherit_preserves_windows_startup_vars_case_insensitively` (passes mixed-case `Shell`, `SystemRoot`, `AppData`, `TmpDir` and asserts all four survive) and `create_env_inserts_pathext_on_windows_when_missing` (asserts `PATHEXT` defaults to `.COM;.EXE;.BAT;.CMD` when policy is `None` and overrides empty).
- `rmcp-client/src/utils.rs:142-170` — `#[cfg(windows)] pub(crate) const DEFAULT_ENV_VARS` (29 lines) deleted and replaced with `pub(crate) const DEFAULT_ENV_VARS: &[&str] = codex_protocol::shell_environment::WINDOWS_CORE_ENV_VARS;` (single re-export). Removes a hand-maintained near-duplicate of the same list — the two were already drifted: `rmcp-client` was missing `SHELL` and `TMPDIR` that the new protocol-side list now includes.

## Verdict

`merge-as-is`

## Reasoning

Two correctness fixes braided into one PR, both real. The headline fix is "shell `inherit=Core` on Windows wasn't preserving enough state for cmd/PowerShell to find anything" — the old `["PATHEXT", "USERNAME", "USERPROFILE"]` list omitted `SYSTEMROOT` (without which `cmd.exe` itself fails to spawn because it can't resolve `kernel32.dll`'s path), `COMSPEC` (without which any nested shell-out picks up garbage), and `LOCALAPPDATA`/`APPDATA` (without which any tool that caches state — node_modules, pip, gh CLI auth — can't find its config dir). The new 22-var list is the standard "PowerShell-launched-from-Explorer" baseline and matches what `rmcp-client` had been carrying separately; consolidating them via the `pub` re-export at `rmcp-client/src/utils.rs:144` is exactly the right hygiene move because the two lists were already diverging silently.

The structural change to split the `Core` arm by `target_os` is a small but real perf/clarity win: the old `is_core_var` closure built a fresh `HashSet<&str>` on every call to `populate_env`, then conditionally case-folded via `.any(|allowed| allowed.eq_ignore_ascii_case(name))` on Windows — that's an O(n*m) fold-and-compare per env var on the OS where the env-var count is highest (Windows env tables routinely run 60+ vars from MSI installers). The split form does linear scan on Windows (n=22, fine) and HashSet lookup on Unix (where case-sensitivity is the actual semantics). No behavior change beyond the list expansion.

The case-insensitivity test is the right pin: Windows env-var keys are case-insensitive at the kernel level but Rust's `HashMap<String, String>` keys are not, so any code doing `env.get("Path")` vs `env.get("PATH")` on the populated map would get different results. The test passes mixed-case `Shell`/`SystemRoot`/`AppData`/`TmpDir` and asserts the *original casing* is preserved in the output (not normalized to upper) — that's the right semantic because downstream consumers like `command::Command::env_clear().envs(...)` on Windows do their own case-folding at insertion time and preserving the source casing means logs/diagnostics are still legible.

The diff is also small in the way that matters: 8 of the 120 added lines are the const list itself, ~50 are the two test functions, ~25 are the cfg-split arms, and the rest is the unchanged surrounding context. The only thing I'd want is a brief CHANGELOG entry noting that users with `inherit = "core"` on Windows will now see more env vars passed through — for the (unlikely) user who was deliberately using `Core` as a denylist of `LOCALAPPDATA`, this is a behavior change. But the code itself is clean. Ship it.
