# openai/codex #21111 — Warn on invalid config enum values

- Head SHA: `01c513fcaa99f2de4a31dd78866f30f93029328e`
- Diff: +595 / −151 across 16 files (centerpiece: new `codex-rs/config/src/lenient.rs`)

## Verdict: `merge-after-nits`

## What it does

Replaces hard-fail `serde::Deserialize` on ~14 enum-typed config fields with a new `Lenient<T>` wrapper that always succeeds: it tries `T::deserialize` first and on failure stashes the raw `toml::Value` as `Lenient::Invalid(value)`. Downstream consumers explicitly choose to either drop invalids silently (`into_valid()`), drop with a captured warning (`into_valid_with_warning(field_path, &mut warnings)`), or report on the raw bad value (`invalid_value()`). The warnings are then bubbled up through `ConfigLayerStack::with_additional_startup_warnings` so the host (TUI/exec) can surface them once at startup instead of nuking the whole config.

The migration touches every enum field that today causes a config-load fail when a user types `sandbox_mode = "make-it-so"` or `model_reasoning_effort = "extreme"`: `approval_policy`, `approvals_reviewer`, `sandbox_mode`, `forced_login_method`, `cli_auth_credentials_store`, `mcp_oauth_credentials_store`, `file_opener`, `model_reasoning_effort`, `plan_mode_reasoning_effort`, `model_reasoning_summary`, `model_verbosity`, `personality`, `service_tier`, `experimental_thread_store`, `web_search` (in `config_toml.rs:107-358` and the parallel set in `profile_toml.rs:25-65`).

The behavioral pivot is captured cleanly in the renamed test `invalid_user_value_written_if_overridden_by_managed` at `app-server/src/config_manager_service_tests.rs:458-491`: previously the loader rejected `approval_policy = "bogus"` even when a managed-config layer supplied a valid override; now the write succeeds, the bogus value is preserved on disk (`"model = \"user\"\napproval_policy = \"bogus\""`), and the *effective* config is still valid because the managed layer wins.

## Why the design is sound

1. **The `Lenient` wrapper is a clean primitive, not a hack.** `lenient.rs:11-109` is ~100 lines: an enum, a private `LenientInput<T>` mirror with `#[serde(untagged)]` for the deserializer (which is the standard idiom for "try this, else fall back"), `Serialize` that round-trips both branches verbatim, and a `JsonSchema` impl that delegates to `T` so the public schema doesn't leak the sloppiness. The `From<T> for Lenient<T>` impl makes call-site updates ergonomic — see how `personality_migration` tests update from `Some(Personality::Pragmatic)` to `Some(Lenient::from(Personality::Pragmatic))` in one obvious line at `app-server/tests/suite/v2/turn_start.rs:1327`.
2. **The downstream `into_valid`/`into_valid_with_warning` split is the right contract.** Some sites (the `UserSavedConfig` round-trip in `config_toml.rs:512-525`, the `Profile` projection in `profile_toml.rs:81-92`) genuinely want silent drop because they're projecting the parsed struct into another wire format and an "invalid value" entry there would be nonsense. Other sites (the new test at `core/src/config/config_loader_tests.rs:121-200` showing `into_valid_with_warning("sandbox_mode", &mut enum_warnings)`) want the warning surfaced. Forcing the choice at the call site is better than baking either policy into `Lenient` itself.
3. **The signature-changing helper `apply_sandbox_workspace_write_overrides` (now `permission_profile`?) at `config_toml.rs:719-734` correctly threads `config_sandbox_mode` as a third explicit parameter** instead of reading it off `&self` — that's because `self.sandbox_mode` is now `Option<Lenient<SandboxMode>>` and the caller needs to do the `into_valid` decision once and pass the *resolved* `Option<SandboxMode>` down. Callers all updated consistently.
4. **The `permission_profile` `sandbox_mode_was_explicit` invariant is preserved.** The old code computed `sandbox_mode_was_explicit = override.is_some() || profile.is_some() || self.sandbox_mode.is_some()`. The new code uses the passed-in `config_sandbox_mode.is_some()` instead — same semantic, because the caller passes `self.sandbox_mode.and_then(Lenient::into_valid)` and an *invalid* value is still semantically "explicit". Wait — actually that's wrong: if the user typed `sandbox_mode = "bogus"`, the old code would treat it as "explicit but parse-failed → whole load errors", and the new code treats it as "not set" for the purposes of `was_explicit`. That's a subtle semantic shift worth a comment in the PR; in practice it makes the failure mode fall back to the default, which is what users want.

## Nits

- **Test name lost a tiny bit of intent.** `invalid_user_value_rejected_even_if_overridden_by_managed` → `invalid_user_value_written_if_overridden_by_managed` reads as "we now write the bogus value to disk because the managed layer overrides it." The behavior is correct (we deliberately preserve the user's typo so they can fix it without losing data), but the test would be clearer with a comment line above the rename explaining *why* — future readers will wonder "wait, isn't writing `approval_policy = "bogus"` to disk a bug?" The expected_string `"model = \"user\"\napproval_policy = \"bogus\""` (line 489) is the proof, but it's load-bearing.
- **`Lenient::Invalid(TomlValue)` retains the original parse value, but the warning text only includes `field_path` and `value`** (`lenient.rs:34`). For nested fields this is fine, but for enum mismatches the value is just the raw string `"bogus"` — *what* it should have been is not surfaced. Consider including `T`'s expected variants in the warning when feasible (e.g., via `serde::Deserialize` reflection or a hand-maintained `T::expected_values()` helper). Probably a follow-up.
- **Schema delegation drops the `Invalid` branch from public schema.** `JsonSchema` for `Lenient<T>` returns `T::json_schema(generator)` (`lenient.rs:103-108`), so consumers introspecting `config_schema.json` see `approval_policy: AskForApproval` (the enum) and have no way to know the loader will tolerate non-enum values. That's actually the *desired* behavior — schema clients shouldn't be encouraged to write garbage — but a comment explaining the deliberate asymmetry would help. The runtime is lenient; the contract is strict.
- **`with_additional_startup_warnings` early-returns on empty input** (`state.rs:215-217`), which is fine, but the `get_or_insert_with(Vec::new).extend(warnings)` pattern allocates an `Option<Vec<String>>` slot even when no warnings ever fire. Negligible.
- **`exec/src/lib.rs` and `tui/src/lib.rs` were touched** (per the file list) but I didn't see those hunks in the truncated diff. Worth confirming both binaries actually surface the new `startup_warnings` to the user — otherwise the warnings are collected and dropped, which would silently regress UX vs. the old "load fails loudly" behavior.

## Citations

- `codex-rs/config/src/lenient.rs:1-109` — new `Lenient<T>` wrapper, `LenientInput<T>` deserializer, `into_valid_with_warning` helper
- `codex-rs/config/src/config_toml.rs:107-358` — 14 enum fields wrapped
- `codex-rs/config/src/profile_toml.rs:25-65` — parallel wrap on profile-scoped fields
- `codex-rs/config/src/state.rs:215-223` — `with_additional_startup_warnings` extend-or-insert
- `codex-rs/app-server/src/config_manager_service_tests.rs:458-491` — renamed test pinning the new "write-with-managed-override" semantic
- `codex-rs/core/src/config/config_loader_tests.rs:121-200` — end-to-end test that `sandbox_mode = "make-it-so"` produces a warning, not an error
- `codex-rs/config/src/config_toml.rs:719-734` — `permission_profile` signature change threading `config_sandbox_mode` explicitly
