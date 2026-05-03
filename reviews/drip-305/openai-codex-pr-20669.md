# openai/codex PR #20669 — Prepare selected environment plumbing

- **Author:** starr-openai
- **Head SHA:** `3cbeededde7466ab30fbb37144be699dd86a2812`
- **Base:** `main`
- **Size:** +155 / -123 (multi-file refactor)
- **Stack:** middle PR of a 3-PR stack (#20646 → **#20669** → #20647)

## What it does

Prep refactor for multi-environment process tools. Three things:

1. Keeps `ResolvedTurnEnvironments` wrapped end-to-end through `TurnContext`
   instead of unwrapping it back to a `Vec<TurnEnvironment>`.
2. Adds `TurnContext::resolve_path_against` as a single helper for
   cwd-relative path resolution.
3. Replaces the boolean `tools_config.has_environment` with a tri-state
   `ToolEnvironmentMode::{None, Single, Multiple}`.

## Diff observations

- `codex-rs/core/src/codex_delegate.rs:33,96` — drops the local
  `ResolvedTurnEnvironments` import and stops re-wrapping
  `parent_ctx.environments` because the field is now already
  `ResolvedTurnEnvironments`.
- `codex-rs/core/src/environment_selection.rs:27-49` —
  `ResolvedTurnEnvironments` gets `Default`, and `primary_turn_environment` is
  renamed to `primary()`. `primary_environment` and `primary_filesystem` are
  rewired to call the shorter name.
- `codex-rs/core/src/context/environment_context.rs:178-181`,
  `codex-rs/core/src/session/mcp.rs:224`, `codex-rs/core/src/session/session.rs:971`
  — every callsite that used `turn_context.primary_environment()` /
  `&turn_context.environments` now reaches through
  `turn_context.environments.primary()` /
  `turn_context.environments.turn_environments`.
- `codex-rs/core/src/session/review.rs:55` — `with_has_environment(...)`
  becomes `with_environment_mode(...)` and reads
  `parent_turn_context.tools_config.environment_mode`.
- `codex-rs/core/src/session/tests.rs:2944-4452` — test helper
  `turn_environments_for_tests` is updated to return
  `ResolvedTurnEnvironments` directly; assertions move from
  `environments.len()` / `environments[0]` to
  `environments.turn_environments.len()` / `environments.turn_environments[0]`,
  and the single `primary_environment()` call goes through `.environments`.

## Strengths

- Mechanical, type-driven rename + wrapping change; LSP/`cargo check` will
  catch any missed callsite. The grep-able rename
  (`primary_turn_environment` → `primary`) is consistent across module,
  callers, and tests.
- Tri-state `ToolEnvironmentMode` is a clean strict-improvement over a bool
  for "0 / 1 / N environments" — exactly the kind of thing that becomes ugly
  if it stays a bool past the next PR.
- Tests are updated in lockstep, including
  `absolute_cwd_update_with_turn_environment_is_allowed` and
  `turn_environments_set_primary_environment` which now assert against the
  wrapped struct.

## Concerns

- "Tests not run locally; covered by GitHub CI for the stack" — please make
  sure the stack's CI run on this exact head SHA actually went green before
  merging; with a 4400+ line test file touched, even one missed callsite is a
  visible compile break.
- `ResolvedTurnEnvironments::Default` is added but the diff doesn't show a
  caller that needs it. If it's only there to make the wrapper construct
  cleanly somewhere downstream in #20647, fine — otherwise it can wait.
- `primary()` is short and ambiguous outside the module; consider `pub(crate)
  fn primary_turn(&self)` or doc'ing it on the struct so readers don't have
  to chase the type.

## Nits (non-blocking)

- A few callsites went from `turn_context.primary_environment()` (one method
  call) to `turn_context.environments.primary()` (two). For the next PR, an
  inherent `TurnContext::primary_environment(&self)` shim could keep the
  call-site noise down without losing the wrapper type.

## Verdict

**merge-after-nits** — refactor is sound, but please verify CI is green on
`3cbeededde7466ab30fbb37144be699dd86a2812` before landing, and ideally rename
`primary()` → something less ambiguous in a follow-up.
