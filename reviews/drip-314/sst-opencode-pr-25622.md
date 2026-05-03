# Review — sst/opencode #25622: configurable context cost display

- **PR:** https://github.com/sst/opencode/pull/25622
- **Head SHA:** `d8a72fb88869b8937180197d19b8a1a6fab6c6f0`
- **Author:** Arjith8
- **Files:** 5 changed (+54 / −12)
  - `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx` (+7/−1)
  - `packages/opencode/src/cli/cmd/tui/config/tui-schema.ts` (+8/−0)
  - `packages/opencode/src/cli/cmd/tui/feature-plugins/sidebar/context.tsx` (+17/−9)
  - `packages/web/src/content/docs/config.mdx` (+9/−1)
  - `packages/web/src/content/docs/tui.mdx` (+13/−1)

## Verdict: `merge-after-nits`

## Rationale

The change adds two opt-in config blocks (`session_bar.{context,cost}` and `sidebar.{context,cost}`) that gate the existing usage display in the prompt's status row and the sidebar's context box. Defaults are preserved (`?? true`) so existing users see no behaviour change — the rollout is strictly additive. Schema changes in `tui-schema.ts:27-34` mirror the call sites in `prompt/index.tsx:1457-1460` and `sidebar/context.tsx:36-51`, and the docs (`config.mdx`, `tui.mdx`) are updated in lockstep, which is the right hygiene for a TUI surface that's user-facing.

Two nits worth flagging before merge. First, in `sidebar/context.tsx:38`, the outer `<Show when={(tuiConfig.sidebar?.context ?? true) || (tuiConfig.sidebar?.cost ?? true)}>` collapses the entire `<box>` when both are off, which is fine, but the inner two `<Show>` blocks each re-evaluate the same accessors. Pulling `const showContext = ...; const showCost = ...;` out once would make the intent (and the tracked-signal surface) clearer. Second, when `sidebar.context` is false but `sidebar.cost` is true, the heading "Context" is hidden but the cost line remains on its own with no label — that's probably fine but worth eyeballing visually before shipping.

The schema design is consistent with the surrounding `TuiOptions` shape (top-level optional groups with required boolean members, default `true`), and the docs additions in `tui.mdx:386-389` document each leaf key separately, which matches the existing style there. No tests added, but the change is purely conditional rendering driven by config, so I'd accept this as-is once the two nits above are eyeballed.

## Banned-string check

Diff scanned; no banned tokens present.
