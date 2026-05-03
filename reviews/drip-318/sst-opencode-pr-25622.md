# Review — sst/opencode#25622 — feat(opencode): configurable context cost display

- PR: https://github.com/sst/opencode/pull/25622
- Head: `d8a72fb88869b8937180197d19b8a1a6fab6c6f0`
- Verdict: **merge-after-nits**

## What the change does

Adds `tui.session_bar.{context,cost}` and `tui.sidebar.{context,cost}` toggles
to `tui-schema.ts`, then short-circuits the prompt session bar render
(`prompt/index.tsx` ~L1454-1460) and the sidebar context view
(`feature-plugins/sidebar/context.tsx`) on those flags. Defaults are `true`
so existing behavior is preserved.

## Strengths

- Schema + UI + docs (`packages/web/src/content/docs/config.mdx`,
  `tui.mdx`) updated together, so the option is discoverable.
- Defaults are `true` everywhere via `?? true`, preserving today's UX.
- Sidebar wraps the whole `<box>` in a `<Show when={…context || …cost}>`,
  so when the user disables both they don't get a blank decorative box.

## Nits / asks

1. Slight asymmetry: in `prompt/index.tsx` the session-bar branch only checks
   `session_bar.{context,cost}`, but for the sidebar there's also a
   surrounding `<Show>`. The session bar should similarly hide its outer
   `<text>` when both fields render `undefined` after `.filter(Boolean)`,
   otherwise an empty `<text>` line still renders.
2. Inside the new `z.object({ context: z.boolean().default(true), … })`:
   because the parent is `.optional()`, the `.default(true)` on inner fields
   only fires when the object is present. That's fine, but document that
   omitting `session_bar` entirely keeps both visible — which you already
   model in code via `?? true`. Worth a one-line comment in the schema.
3. The new fields belong in a single shared shape (e.g.
   `const StatBarToggles = z.object({...})`) since `session_bar` and
   `sidebar` have identical schemas — easier to extend later.

No blockers; ship after addressing the empty-`<text>` nit at minimum.
