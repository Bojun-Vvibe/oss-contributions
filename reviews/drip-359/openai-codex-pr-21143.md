# openai/codex PR #21143 — Support multi-env view_image routing

- Repo: `openai/codex`
- PR: #21143
- Head SHA: `a09589641ea9b8c3b957f89462cb57a359d7a61d`
- Author: `starr-openai`
- Updated: 2026-05-05T04:27:55Z
- Verdict: **merge-after-nits**

## What it does

Brings `view_image` in line with the rest of the multi-environment tool family (already done for `read_file`, `list_dir`, `apply_patch`, `shell`). Two-part change:

1. **`codex-rs/tools/src/view_image.rs`** — adds `include_environment_id: bool` to `ViewImageToolOptions` and calls the shared `maybe_insert_environment_id_parameter(&mut properties, options.include_environment_id)` so the `environment_id` JSON-Schema property is added to the tool spec only when multi-env is active. Also drops the word "Local" from the tool description and the `path` parameter description (`"Local filesystem path to an image file"` → `"Filesystem path to an image file"`) since the path is now resolved against the chosen environment's `cwd`, which may be remote.

2. **`codex-rs/core/src/tools/handlers/view_image.rs:74-117`** — replaces the `turn.environments.primary()` direct lookup with the shared `resolve_tool_environment(turn.as_ref(), &arguments)?` helper, then computes `abs_path = cwd.join(path)` against the resolved environment's `cwd` rather than `turn.resolve_path(...)`. The remote-sandbox guard becomes `target_environment.environment.is_remote().then(|| turn.file_system_sandbox_context_for_cwd(&cwd, /*additional_permissions*/ None))`.

3. **`tool_registry_plan_tests.rs:691-734`** — adds `view_image_spec_includes_environment_id_only_for_multiple_selected_environments` mirroring the existing `list_dir_spec_includes_environment_id_only_for_multiple_selected_environments` test: builds specs with default (single-env) `ToolsConfig`, asserts `environment_id` absent; flips to `ToolEnvironmentMode::Multiple`, asserts `environment_id` present.

## Strengths

- Reuses the established `resolve_tool_environment` + `maybe_insert_environment_id_parameter` + `file_system_sandbox_context_for_cwd` triplet — same shape as the `list_dir` rollout, so reviewer cost is low and the new behavior is self-consistent.
- Test mirrors the sister-tool test exactly (same `assert_process_tool_environment_id` helper, same single-vs-multi env_mode switch), which keeps the multi-env test surface uniform.
- Description tweak (`Local` → empty, `Local filesystem path` → `Filesystem path`) is the correct wording change for a tool that can now operate against a remote env's filesystem; the existing `code_mode_augments_builtin_tool_descriptions_with_typed_sample` test was updated in lockstep at `tool_registry_plan_tests.rs:2189-2235`.

## Nits

1. **`detail` destructure ordering.** The new `let ViewImageArgs { path, detail } = args;` at `view_image.rs:77` happens *before* the detail is validated against `Some("original")` at line 80, which is fine, but the destructure shadows the outer `detail` field and then reuses the binding name for the parsed `Option<ViewImageDetail>` two lines later. Mild readability nit — rename one of them (`let detail_opt = match detail.as_deref() { ... }`) or keep `args.detail.as_deref()` inline.

2. **No integration test for path resolution against a non-primary environment.** The new test only verifies the *spec* surface (env_id parameter present/absent). There is no test that actually invokes `ViewImageHandler::handle` against a `ToolInvocation` carrying `environment_id` for a non-primary env and asserts the resolved path lands under that env's `cwd`. A `tests/suite/`-level test would catch a future regression where `resolve_tool_environment` silently falls back to primary.

3. **Error message preservation.** The `view_image is unavailable in this session` message at line 79 is reused verbatim from the old `turn.environments.primary()` path. With multi-env routing, this message can now also mean "the requested `environment_id` doesn't match any selected environment" — a more specific message ("requested environment is not available") would help users distinguish the two failure modes.

## Verdict
**merge-after-nits** — strictly-additive multi-env wiring that mirrors prior tool rollouts; pick up the duplicate-`detail`-binding rename and consider an end-to-end test for the multi-env routing path before landing.
