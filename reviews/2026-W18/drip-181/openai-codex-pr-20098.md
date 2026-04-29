# openai/codex#20098 — fix: ignore dangerous project-level config keys

- **Repo**: openai/codex
- **PR**: [#20098](https://github.com/openai/codex/pull/20098)
- **Head SHA**: `726716f4037fe14d62c0ea7442b4b386581511a1`
- **Author**: owenlin0
- **Diff stats**: +181 / −5 (4 files)

## What it does

Closes a real supply-chain hole: any `.codex/config.toml` checked into a
repository can previously set `openai_base_url`, `chatgpt_base_url`,
`model_provider`, `model_providers`, `profile`, `profiles`, or
`experimental_realtime_ws_base_url` — meaning cloning a malicious repo
could silently redirect the user's OpenAI/ChatGPT credentials to an
attacker-controlled endpoint on the next `codex` invocation.

Fix introduces a `PROJECT_LOCAL_CONFIG_DENYLIST` and runs incoming
project-layer TOML through `sanitize_project_config` before it joins the
layered config. User, system, managed, and runtime layers are unaffected.
A new `ignored_project_config_keys` field on `ConfigLayerEntry` records
what was stripped, for observability/UI.

## Code observations

- `codex-rs/config/src/loader/mod.rs:50-60` — denylist as a
  module-level `&[&str]`. Spec is clear and the doc comment "Project-local
  config comes from repository contents, so it should not get to choose
  where a user's OpenAI or ChatGPT credentials are sent" is exactly the
  right framing for future maintainers. Worth pinning a CVE-style ID or
  GHSA reference here so the next reviewer doesn't re-litigate the list.
- `:721-735` `sanitize_project_config` — clean: clones into a `mut`,
  walks `as_table_mut()`, removes each denylisted top-level key, returns
  `(config, ignored_keys)`. `as_table_mut()` returning `None` (root not a
  table) safely no-ops, which is correct for a malformed file that should
  be rejected elsewhere.
- `:998-1003` — `sanitize_project_config` is called *after*
  `resolve_relative_paths_in_config_toml`. Order matters: if a future
  denylisted key were path-typed, a malicious `..` traversal could be
  resolved before being stripped. None of the current denylist entries
  are path-shaped, but worth a code comment locking the order.
- `codex-rs/config/src/state.rs:67,79,91,107` — `ignored_project_config_keys:
  Vec<String>` plumbed through all three constructors of
  `ConfigLayerEntry`. Boilerplate-heavy but correct; a `#[derive(Default)]`
  on the field would let callers omit it but the explicit-everywhere shape
  is preferred in security-sensitive layered config.
- `codex-rs/core/src/config/config_loader_tests.rs:1646-1727` — new
  `project_layer_ignores_unsupported_config_keys` test covers exactly the
  attack: a project TOML setting all 7 denylisted keys plus a benign
  `model = "project-model"`. Asserts the benign key survives in
  `effective_config()` AND that all 7 denylisted keys appear in
  `ignored_project_config_keys` in the **declared order**. Order assertion
  is brittle if the iterator order changes — recommend `assert_eq!` on a
  sorted comparison or a `HashSet` to avoid pinning iteration order
  unnecessarily.
- The denylist is **top-level keys only**. A nested
  `[some_existing_section.openai_base_url]` would not be touched, but
  none of the codex code paths read such a nested key today. A comment
  to that effect ("denylist is top-level only; nested redefinitions are
  unreachable in current code") would future-proof.

## What's missing

- **No user-facing surfacing** of `ignored_project_config_keys`. Silent
  stripping is the right *security* default, but a one-line warning
  (`tracing::warn!`) on session start when the project layer has any
  ignored keys would let the user notice a malicious repo trying to
  redirect them. The PR plumbs the data structure but leaves the warning
  for a follow-up.
- **No test for the disabled-project-layer path** at `:984-989` — when
  `disabled_reason` is `Some(_)` the project config never enters
  `sanitize_project_config` at all (the empty table is constructed
  directly). That's correct behavior (a disabled layer contributes
  nothing), but a regression test pinning that semantic would help.
- The seven denylisted keys are the right set *today*. Worth a
  follow-up audit to enumerate every config key that affects credential
  routing or remote endpoints (e.g. future
  `apps_mcp_path` from #20231 if any apps-mcp endpoint becomes
  credentialed) so this list stays in sync.

## Verdict: `merge-after-nits`

This is a real and well-shaped fix. Nits:

1. Surface a `tracing::warn!` (or one-line CLI hint) when
   `ignored_project_config_keys` is non-empty, citing the keys, so a
   silently-redirected user notices on first launch. Without this the
   correct fix produces an indistinguishable UX from "the malicious
   config worked".
2. Replace the order-sensitive `assert_eq!` on `ignored_project_config_keys`
   at `:1696-1706` with an order-agnostic comparison (sort both sides, or
   use a `HashSet`) so a future iteration-order change doesn't break the
   test for cosmetic reasons.
3. Add a test covering "project TOML has the denylisted key inside
   `[profiles.foo]`" to lock in that nested redefinitions inside a
   denylisted section are also stripped via the parent-key removal.
4. Inline a doc comment at `:1003` documenting the
   sanitize-after-resolve-relative-paths ordering invariant so a future
   reorder doesn't silently reintroduce path-traversal exposure if a
   denylisted key ever becomes path-typed.
5. Cross-reference this change in the security advisory / release notes —
   users on prior versions should be encouraged to audit recent project
   configs for the seven denylisted keys.

## Follow-ups

- Audit `codex_apps_mcp_url_for_base_url` (introduced in #20231) and
  any other endpoint-resolution helpers for project-config redirect
  surfaces.
- Consider an opt-in escape hatch (e.g. `--trust-project-config-redirects`
  CLI flag, or a per-project trust grant) for advanced users who genuinely
  want a project to set its own `model_provider`.
