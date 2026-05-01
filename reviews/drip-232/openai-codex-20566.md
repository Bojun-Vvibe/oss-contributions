# openai/codex#20566 — [tool_suggest] More prompt polishes

- **Author:** Matthew Zeng (mzeng-openai)
- **Head SHA:** `16bfbf80ddc7fce4f1c1a5c26a40da56011f301a`
- **Base:** `main`
- **Size:** +14 / -14 across 6 files
- **Files changed:** `codex-rs/core/templates/search_tool/tool_suggest_description.md`, `codex-rs/core/tests/suite/tool_suggest.rs`, `codex-rs/tools/README.md`, `codex-rs/tools/src/tool_discovery.rs`, `codex-rs/tools/src/tool_discovery_tests.rs`, `codex-rs/tools/src/tool_registry_plan_tests.rs`

## Summary

Two-axis polish to the `tool_suggest` discoverable-tool surface to reduce model misfires. (a) Renames the tool from `tool_suggest` → `request_plugin_install` at all 6 source-of-truth sites (constant in `tool_discovery.rs`, suite-test constant, README mention, two unit-test arms, and the model-facing prose in the description template). (b) Rephrases the description's English from "suggestion" to "install" wording so the model interprets the tool as an *action* it's asking the user to take rather than something passive it can keep "suggesting". The PR body explicitly notes this fixes a misfire mode where the model was firing `tool_suggest` even when it should have called `tool_search` first.

## Specific code references

- `tool_discovery.rs:18`: the load-bearing constant flip — `pub const TOOL_SUGGEST_TOOL_NAME: &str = "request_plugin_install"`. Note the variable name itself is *not* renamed (still `TOOL_SUGGEST_TOOL_NAME`), only its value — slightly inconsistent (callers will read `TOOL_SUGGEST_TOOL_NAME` and see `"request_plugin_install"` at the value site), but the public-API change is just the wire-string the model sees, and keeping the Rust identifier stable avoids a much larger downstream rename. Defensible trade-off; worth a `// renamed at the wire level only` comment for the next maintainer.
- `tool_suggest_description.md:1`: title flipped from "Tool suggestion discovery" to "Request plugin/connector install" — anchors the rest of the prose in action-verb framing rather than nouning "suggestion".
- `tool_suggest_description.md:10`: `Do not use tool suggestion for adjacent capabilities` → `Do not use this tool for adjacent capabilities`. Removes the noun "tool suggestion" which the model was apparently treating as a separate concept ("oh, I should *suggest*") rather than the action being gated.
- `tool_suggest_description.md:16`: workflow rule 1 tightens the precondition: old prose said "If `tool_search` is available, call `tool_search` before calling `tool_suggest`. Do not use tool suggestion if the needed tool is already available, found through `tool_search`, or callable after discovery." New prose: "If current active tools aren't relevant and `tool_search` is available, only call this tool after `tool_search` has already been tried and found no relevant tool." The new shape pins the *order* (tool_search first, then this tool only on miss) more aggressively, and the "only call this tool after `tool_search` has already been tried and found no relevant tool" clause closes the loophole where the model was firing this tool *in parallel* with `tool_search` rather than serially after.
- `tool_suggest.rs:24` and `tool_discovery_tests.rs:65, :101` and `tool_registry_plan_tests.rs:1875`: the test-side string assertions are correctly updated to expect the new wire-string `request_plugin_install` and the new prose. These are the load-bearing pins — without these updates the rename would be silently regressed by any future PR that re-flipped the constant.

## Reasoning

This is the right shape for an LLM-facing prompt fix: identify the misfire (model calling `tool_suggest` when `tool_search` would suffice), rename the tool to remove the implied noun-form trigger ("suggestion" → action verb "install"), and tighten the workflow prose to make the precondition explicit at the order-of-operations level. The 6-file fan-out is the correct breadth — every site that mentions the wire name is updated atomically, and the test arms pin the new shape.

Two nits: (a) the Rust identifier `TOOL_SUGGEST_TOOL_NAME` left holding the value `"request_plugin_install"` will confuse the next reader; either rename the identifier too or add an inline comment; (b) workflow rule 3 still says "If we found both connectors and plugins to suggest, use plugins first" — the verb "suggest" wasn't fully scrubbed there, slight inconsistency with the rest of the action-verb pivot. (c) The README at `tools/README.md:30` updates the bullet to mention `request_plugin_install` but the surrounding `discoverable-tool models, client filtering, and ToolSpec builders for tool_search and request_plugin_install` reads slightly oddly with the new name — minor copy polish opportunity.

## Verdict

**merge-after-nits**
