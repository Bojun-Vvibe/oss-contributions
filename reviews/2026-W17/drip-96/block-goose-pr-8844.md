# block/goose #8844 — fix: convert quoted numeric config values to numbers if needed

- Author: philipphenkel
- Head SHA: `1a18189ff8cfd1651a8062e67b29a836ff65042b`
- +61 / −3 across `crates/goose/src/config/base.rs` (+61/-3) — single-file change
- PR link: <https://github.com/block/goose/pull/8844>

## Specifics

- Real-user-reported bug from issue #8437: writing `OLLAMA_TIMEOUT: '6000'` into `~/.config/goose/config.yaml` (which is what the goose desktop app does — quoted strings) caused the typed read `config.get_param::<u64>("OLLAMA_TIMEOUT")` to fail because `serde_yaml::from_value` won't coerce a YAML string scalar to a `u64`. Same applies to `OPENAI_TIMEOUT` and `LITELLM_TIMEOUT` — all the provider-timeout reads were silently ignoring their config values when stored as quoted strings.
- Fix at `base.rs:732-748` adds a fallback path: try `serde_yaml::from_value(value.clone())` first; if that fails AND `value.as_str()` is `Some(string_value)`, run it through `Self::parse_env_value(string_value)` (the same scalar parser used for `$ENV_VAR` indirection elsewhere in this file) and then `serde_json::from_value` to coerce. If the original value is not a string (e.g. a real bool/list), or if parse-env fails, return the original `yaml_err`. That's the correct error-precedence: the original deserialization error is what the caller actually wanted to see in the failure case.
- Critical correctness check: the `Ok(value) => Ok(value)` arm at `base.rs:741` means the *normal* YAML deserialization path is unchanged. A `OLLAMA_TIMEOUT: 1200` (unquoted) still hits the fast path. Only the failure case dips into the parse-env fallback. Zero regression risk for the existing well-formed-config users.
- String-callers preserved: test `test_get_param_reads_quoted_numeric_yaml_as_string` at `base.rs:+1198-1207` asserts that `config.set_param("XXX_TIMEOUT", "300")` followed by `config.get_param::<String>("XXX_TIMEOUT")` returns `"300"` (not `300_u64.to_string()`). This is the right invariant — when the caller explicitly asks for `String`, the YAML→String deserialization succeeds on the fast path and the fallback is never reached. The fallback only fires when the typed target is numeric (or any other type that fails to deser-from-string).
- Test `test_get_param_rejects_invalid_string_as_u64` at `base.rs:+1212-1221` confirms that `XXX_TIMEOUT: "invalid"` still returns `Err(ConfigError::DeserializeError(_))` — i.e. the fallback doesn't paper over genuine garbage. The chain is: `serde_yaml` fails on `"invalid"` → `value.as_str() = Some("invalid")` → `parse_env_value("invalid")` either succeeds with a string-shaped JSON value (`Value::String("invalid")`) → `serde_json::from_value::<u64>(Value::String("invalid"))` fails → `.map_err(|_| yaml_err.into())` returns the *original* yaml error. Good error preservation.
- Test coverage matrix (`base.rs:+1177-1221`):
  - `test_get_param_reads_numeric_yaml_as_u64`: unquoted numeric → u64 (fast path)
  - `test_get_param_reads_quoted_numeric_yaml_as_u64`: quoted numeric → u64 (new fallback path) — the actual bug fix
  - `test_get_param_reads_quoted_numeric_yaml_as_string`: quoted numeric → String (fast path preserved)
  - `test_get_param_rejects_invalid_string_as_u64`: garbage string → typed error preserved (fallback rejects correctly)

## Concerns

- The fallback re-uses `parse_env_value`, which is named for env-var parsing but is being used here for YAML-string-scalar parsing. They're behaviorally similar (both turn a string into a serde_json::Value via best-effort number/bool/string detection) but the function name now lies a bit. A short comment at `base.rs:740` saying "reuse env-var scalar parser for the fallback because YAML-as-string semantics match $VAR semantics" would help the next reader.
- The fallback only fires on `value.as_str()` being `Some` — a YAML *block scalar* with newlines (e.g. literal-block `|` or folded `>` styles) would still produce a string, so the parser handles them, but a numeric stored as a YAML *sequence* (`OLLAMA_TIMEOUT: [6000]` — a config-file mistake) would fail with the original `yaml_err`, which is the right behavior.
- `parse_env_value` failure mode at `base.rs:741`: if `parse_env_value(string_value)` itself returns `Err`, the `?` will propagate that error up — but the caller asked for the *typed* read, and what they get back is now a `ConfigError` from the env-parser, not the original yaml deser error. Reviewers should confirm `parse_env_value` returns the same `ConfigError` flavor as `yaml_err` would; otherwise error messages may shift in confusing ways for unrelated failure modes.
- No integration test against an actual `OLLAMA_TIMEOUT: '6000'` round-trip from the desktop-app code path. The 4 unit tests cover the deserialization mechanic but not the end-to-end "desktop app writes quoted, provider reads u64" loop. That's a fair scope cut for a one-file fix; a follow-up could add the integration coverage.
- The desktop-app side (which writes the quoted string) isn't fixed here. Long-term the right answer might be "make the desktop app write proper YAML scalars" so the fast path always works, with this fallback as the safety net for already-deployed config files. Worth a TODO referencing the desktop-app issue.

## Verdict

`merge-as-is` — the fix is minimal (single-file, +61/-3, four tests covering the full matrix), preserves backward compat (fast path unchanged), preserves error semantics (original yaml_err returned on fallback failure), and addresses a confirmed real-user issue from #8437 with a documented user-side workaround that this PR removes. The naming nit on `parse_env_value` and the desktop-app-side follow-up are both fine as separate work. Good defensive change.
