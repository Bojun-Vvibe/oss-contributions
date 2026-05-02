# sst/opencode PR #25363 — feat(opencode): Switching agents considers the agent's configured model variant

- URL: https://github.com/sst/opencode/pull/25363
- Head SHA: `8e34f73be507c29ac84ece7e67072f9acc6e4fbd`
- Author: lowlyocean
- Verdict: **merge-after-nits**

## Summary

Extends the existing "auto-update model when agent changes" effect in `packages/opencode/src/cli/cmd/tui/context/local.tsx` so that, after the providerID/modelID is set from the agent definition, the agent's configured `variant` (if any) is also propagated into `model.variant`. Without this, a user switching to an agent whose definition pins a model variant (e.g. `claude-sonnet:thinking`) would silently fall back to whatever variant the previous agent had selected.

## Line-level observations

- `packages/opencode/src/cli/cmd/tui/context/local.tsx` lines ~397–414: the `if (isModelValid(value.model))` branch is converted from a single-statement `if` to a block. Inside the block, after `model.set({...})` is called, the new line `if (value.variant) model.variant.set(value.variant)` is added. The `else` branch (toast warning for invalid model) is preserved verbatim, just dangling off the new `}`.
- The order of operations matters: `model.set(...)` first, then `model.variant.set(...)`. If `model.set` internally resets the variant (a common pattern for "reset dependent state on root change"), this ordering is the only correct one. Reviewer should confirm `model.set` does not asynchronously clear variant after this synchronous follow-up.
- The new `if (value.variant)` only writes when truthy — so an agent that explicitly omits `variant` will *inherit* the previously-active variant rather than reset it. That may or may not be intended. If the goal is "agent fully owns model+variant", this should arguably `else model.variant.set(undefined)` so switching from a thinking-variant agent to a non-thinking agent clears the bit.

## Suggestions

1. Decide explicitly whether absence of `value.variant` should clear or preserve the existing variant; document the choice with a one-line comment so the next maintainer doesn't "fix" it the other way.
2. Add a TUI-level test (or at minimum a unit test against the `LocalProvider` effect) that switches between two agents — one with `variant: "thinking"`, one without — and asserts the post-switch state.
3. Minor style: the indentation of the trailing `else` after the new `}` is now inconsistent with the surrounding ternary-ish style. Consider an explicit `else { ... }` block for readability symmetry.
