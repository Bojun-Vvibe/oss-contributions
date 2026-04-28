# openai/codex #19904 — fix: configure AgentIdentity AuthAPI base URL

- URL: https://github.com/openai/codex/pull/19904
- Head SHA: `8e394fafaf732c35f1772239c81ec8533faae2b8`
- Size: +172 / -29 across 18 files
- Verdict: **merge-after-nits**

## What the change does

AgentIdentity task registration was hardcoded to a single AuthAPI base URL,
which made it impossible to point a build at a non-prod AuthAPI without
patching the binary. This PR adds a config knob `agent_identity_authapi_base_url`
and threads it as a function argument through the auth-loading path, with the
explicit non-goal of baking any internal staging URL into the OSS source.

- Config schema:
  - `codex-rs/config/src/config_toml.rs:283-284` adds
    `pub agent_identity_authapi_base_url: Option<String>` to `ConfigToml`
  - `codex-rs/config/src/profile_toml.rs:42` mirrors it on `ConfigProfile` so
    profile overrides work
  - `codex-rs/core/config.schema.json:323-325` and `:2402-2405` regenerate the
    JSON schema (top-level + `ConfigToml` definition) — both with `"type":
    "string"` and the doc string from the Rust struct
- Threading: every constructor / loader of `AuthManager` now accepts an
  additional `agent_identity_authapi_base_url: Option<String>` parameter,
  with `/*agent_identity_authapi_base_url*/ None` at every test-harness
  call site (consistent commenting style — every caller is grep-able).
- Production wiring at `codex-rs/cloud-tasks/src/util.rs:53` reads
  `config.agent_identity_authapi_base_url` and passes it through to
  `AuthManager`.
- New CLI surface: `codex-rs/cli/src/login.rs:365-369` updates
  `from_auth_storage` to accept the new arg, derived from
  `config.agent_identity_authapi_base_url.as_deref()`.

## What is load-bearing

- The default behaviour at `cloud-requirements/src/lib.rs:741` passes `None`
  for `agent_identity_authapi_base_url`, which means the AuthManager keeps its
  prod-default registration URL — no behaviour change for users who never
  configure the new field. That's the right default.
- The threading uses `Option<String>` rather than a struct; with 18 files
  touched and zero tests now failing on the new param, the call-graph is
  simple enough that the extra positional arg is acceptable. A
  `LoadAuthManagerArgs` struct with builder semantics would be cleaner but
  this is a defensible tradeoff for a stack-of-4 PRs.

## Nits

1. **Positional `Option<String>` arg parade**: every test call site now ends
   with `/*chatgpt_base_url*/ None, /*agent_identity_authapi_base_url*/
   None,`. Two adjacent `Option<String>` parameters are easy to swap by
   accident. The `/*comment*/ None` convention is good defense, but a
   typed-newtype (`pub struct AgentIdentityAuthApiBaseUrl(pub String);`)
   would let the compiler catch swap mistakes. Worth a follow-up, not
   blocking.
2. **No test for non-`None` propagation**: every new call site passes `None`.
   There is no test asserting that when
   `config.agent_identity_authapi_base_url = Some("https://staging...")`,
   the value actually reaches `AgentIdentity` task-registration. Recommend
   adding one test in `login/src/auth/auth_tests.rs` (which already gained
   +19/-4 in this PR) that constructs an AuthManager with `Some("...")` and
   asserts the registration target was the configured value, not the prod
   default.
3. **Profile precedence undocumented**: `ConfigProfile` gets the field but
   the merge order between `ConfigToml.agent_identity_authapi_base_url` and
   `ConfigProfile.agent_identity_authapi_base_url` is implicit (presumably
   profile-wins, matching `chatgpt_base_url`). A one-line doc on the field at
   `config_toml.rs:283` saying "Profile-level overrides take precedence" would
   prevent future confusion.

## Risk

- Schema regeneration looks consistent; the field appears once at top-level
  (`:2402-2405`) and once inside `ConfigToml` definition (`:323-325`) which
  is the expected pattern for this codebase.
- The PR sits in a stack (parent #19762 merged, sibling #19763, follow-up
  #19764). The threading shape here will become load-bearing for #19764
  (JWKS verification) — landing this with a propagation test would harden
  the foundation for the JWKS PR.

## Recommendation

Merge after at least one positive test that proves a non-`None`
`agent_identity_authapi_base_url` actually flows to the task-registration
call, plus the one-line precedence doc on the config field.
