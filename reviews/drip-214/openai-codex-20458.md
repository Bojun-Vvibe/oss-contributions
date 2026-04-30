# Review: openai/codex#20458 — [Extension] Allowlist Chrome Extension in the tool_suggest tool

- PR: https://github.com/openai/codex/pull/20458
- Merge SHA: `487716ae74944ed07b3cf660998eb97780012a08`
- Files: `codex-rs/core-plugins/src/lib.rs` (+1/-0)
- Verdict: **merge-as-is**

## What it does

Adds `"chrome@openai-bundled"` to `TOOL_SUGGEST_DISCOVERABLE_PLUGIN_ALLOWLIST`
so the tool-suggest subsystem can discover and surface the Chrome extension
plugin alongside the existing `outlook-calendar`, `linear`, `figma`, and
`computer-use` entries.

## Notable changes

- `core-plugins/src/lib.rs:31` — single new array element appended after
  `figma@openai-curated` and before `computer-use@openai-bundled`. The
  ordering convention in this constant pairs `*-curated` (third-party SaaS
  with vendor-curated metadata) before `*-bundled` (in-process plugins
  shipped with the binary), and the new entry is correctly placed at the
  start of the bundled group.

## Reasoning

This is the same shape as drip-213's #20474 (Canva to suggesteable list) and
several earlier one-line allowlist appends. The constant
`TOOL_SUGGEST_DISCOVERABLE_PLUGIN_ALLOWLIST` is a plain `&[&str]` consumed by
the suggestion subsystem with no precedence ordering, no version pinning,
and no behavioural side effects beyond "may surface to the user as a
suggestion." Adding a string here is the lowest-risk change in the codex
repo — the worst-case failure mode is "user sees a suggestion for something
they don't have installed," which the suggestion subsystem already handles
gracefully.

The `chrome@openai-bundled` namespace is consistent with `computer-use@openai-bundled`,
which is the closest sibling (both are local browser-control plugins
distributed with the binary rather than third-party SaaS connectors). Since
both share the bundled tier, they should share the discovery channel — this
PR closes that asymmetry.

The `[Extension]` prefix in the PR title and the `Chrome Extension` framing
suggest there's a separate plugin crate elsewhere in the workspace
implementing the actual MCP-style server for the Chrome extension; this PR
is just the discovery flag flip and doesn't depend on or block that crate
shipping.

## Nits (non-blocking)

1. The constant has no doc comment explaining the `*-curated` vs
   `*-bundled` distinction or the discovery contract. A 3-line `///` block
   above the constant would help future contributors who land here for the
   first time. Out of scope for this PR.
2. No test pins the array contents. A `#[test] fn allowlist_contains_chrome()
   { assert!(TOOL_SUGGEST_DISCOVERABLE_PLUGIN_ALLOWLIST.contains(
   &"chrome@openai-bundled")); }` would document intent, but the consensus
   in this crate seems to be that one-line constant additions don't warrant
   tests. Fine.
3. If `chrome@openai-bundled` requires a feature flag or capability check at
   the user's tier (e.g. some tier doesn't get bundled plugins), the
   suggestion will surface for users who can't actually install it. Worth
   confirming the tier-gating happens downstream of the allowlist filter
   rather than at it.
