# openai/codex #19881 — unify deferred and normal mcps, register specs for all

- **PR**: [openai/codex#19881](https://github.com/openai/codex/pull/19881)
- **Head SHA**: `258951b6`
- **Stats**: +812/-818 across 24 files in `codex-rs/core/`, `codex-rs/tools/`

## Summary

Refactor that collapses the bifurcated `mcp_tools` / `deferred_mcp_tools`
data flow into a single `HashMap<String, McpToolInput>` where each entry
carries an explicit `defer_loading: bool` flag. `ToolRouter` now
registers specs for *both* direct and deferred tools (previously only
direct tools' specs were registered, which created a discrepancy where
dynamic deferred tools could be invoked but their specs were missing
from `router.specs`). Renames `router.specs()` →
`router.specs_including_deferred()` for clarity, retains
`router.model_visible_specs()` for the LLM-visible subset (filters by
`defer_loading == Some(true)`).

## Specific findings

- `mcp_tool_exposure.rs:18-78` — `build_mcp_tool_exposure` rewritten from
  returning a struct `McpToolExposure { direct_tools, deferred_tools:
  Option<HashMap> }` to returning a flat `HashMap<String, McpToolInput>`.
  The fast-path (small candidate set, no defer) still works (`:38-50`
  builds the map with `defer_loading: false` for every entry); the slow
  path (`:54-76`) explicitly sets `defer_loading: false` for direct
  codex_apps tools and `defer_loading: true` for the rest. Critical
  invariant preserved: direct tools removed from the candidate set
  before iterating remaining → re-inserted with `defer_loading: true`,
  so no tool ends up double-mapped.
- `mcp_tool_input.rs:483-486` — new `pub(crate) struct McpToolInput {
  tool_info: McpToolInfo, defer_loading: bool }`. The `pub(crate)`
  visibility is correct — this is internal plumbing between the
  exposure-policy layer and the router, not a public API surface.
- `tools/router.rs:546-549` — `model_visible_specs` filter changed from
  `.filter(|tool| tool.defer_loading)` (excluding deferred from model
  visibility) to `.filter(|tool| tool.defer_loading != Some(true))`
  pattern at `:599,611`. The shift to `Option<bool>` for the spec-side
  field (vs the input's plain `bool`) is intentional — specs registered
  by non-MCP code paths have no defer-loading concept and should be
  `None` (visible by default). Worth a one-line comment naming this
  asymmetry; right now it reads like a typo.
- `tools/router.rs:569-580` — `specs_including_deferred()` is the new
  full-spec accessor. The rename from `specs()` is the load-bearing
  signal that callers need to think about whether they want the deferred
  set included; making the old `specs()` name disappear forces every
  call site through review.
- `mcp_tool_exposure_test.rs:120-260` — test rewrites use the new
  helper `tool_names_with_defer_loading(&exposed_tools, defer_loading)`
  to assert direct-vs-deferred partitioning through the public
  `HashMap<String, McpToolInput>` shape rather than through the prior
  struct's two fields. Helper is correct (filters by exact match,
  sorts result for deterministic comparison). Coverage retained:
  small-set-direct, large-set-deferred, explicit-app-no-overlap, and
  always-defer-feature paths all still pin their respective behaviors.
- `tool_registry_plan.rs` (in `codex-rs/tools/`) — the spec-registration
  side of the refactor: deferred tools now produce specs at registration
  time (previously skipped), specs carry the `defer_loading: Some(true)`
  marker so the model-visibility filter can drop them. This closes the
  PR's stated goal — `router.specs_including_deferred()` now reports
  every dynamic tool, including deferred ones.

## Nits / what's missing

- Asymmetry between `McpToolInput.defer_loading: bool` (input side) and
  `Spec.defer_loading: Option<bool>` (registry side) reads as
  inconsistent. The `Option` is justified (specs from non-MCP sources
  legitimately have no opinion), but a one-line comment on the spec
  field explaining "None means non-MCP / always visible" would prevent a
  future cleanup PR from collapsing them to plain `bool` and
  accidentally hiding non-MCP specs.
- 24-file refactor with `+812/-818` is right-sized for a single PR (the
  rename `specs()` → `specs_including_deferred()` propagates through
  every caller and splitting that across PRs would leave the tree in an
  intermediate state where the contract is ambiguous), but the PR body
  doesn't call out that this is a behavior change for any external
  consumer of `router.specs()` (none in-tree, but the `pub` visibility
  on the new method suggests external consumers exist).
- No test pins the `model_visible_specs` filter behavior on the
  `Option<bool>` boundary. A test that registers a non-MCP tool spec
  (with `defer_loading: None`) and asserts it lands in
  `model_visible_specs()` output would lock in the "None means visible"
  invariant.

## Verdict

**merge-after-nits** — refactor is correct and closes the stated
discrepancy (deferred tool specs now registered + included in
`specs_including_deferred()`); the data-shape collapse from
two-field-struct to single map with explicit per-tool flag is the right
direction. Two real gaps before merge: (1) comment the
`Option<bool>` vs `bool` asymmetry between the input and spec sides, (2)
add a regression test pinning `model_visible_specs()` behavior on
`defer_loading: None` specs.
