# sst/opencode#25171 — feat(tui): add content_padding option to control horizontal content margins

- **Author:** Aaron Hogue (alohaninja)
- **Head SHA:** `993118cf84f1eecb0b0a09a908672967c60fe465`
- **Base:** `dev`
- **Size:** +10 / -2 across 2 files
- **Files changed:** `packages/opencode/src/cli/cmd/tui/config/tui-schema.ts`, `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx`

## Summary

Surgical TUI ergonomics PR exposing the previously hardcoded horizontal content padding (`paddingLeft={2}`/`paddingRight={2}` and `- 4` width offset) as a `content_padding` config knob bounded `[0, 20]` and defaulting to the prior value of `2` so existing users see zero behavior change.

## Specific code references

- `tui-schema.ts:27-33`: new `content_padding` Zod field — `z.number().int().min(0).max(20).default(2)` with describe text. Bounds are sensibly tight (max 20 prevents a misconfig that would zero-out the content width on a typical 80-col terminal). Default of `2` matches the prior literal at `session/index.tsx:1058`, so this is a true zero-regression default.
- `session/index.tsx:177`: `contentWidth` memo is rewritten from `dimensions().width - (sidebarVisible() ? 42 : 0) - 4` to `... - contentPadding * 2`. The `4` → `contentPadding * 2` substitution preserves the prior arithmetic exactly when `contentPadding === 2` (since `2 * 2 = 4`), which is the load-bearing invariant for layout symmetry between the rendered `<box>` margins and the width budget passed to children that compute their own wrap widths.
- `session/index.tsx:1059`: `paddingLeft={contentPadding} paddingRight={contentPadding}` — the box-render side of the same change. Keeping the `paddingLeft`/`paddingRight` pair coupled to one variable (rather than two independent fields) is the right call: asymmetric horizontal padding would desync from the `* 2` in the `contentWidth` memo and produce off-by-N wrapping bugs.

## Reasoning

Two-line behavior change with one new schema field. The pair (width-budget, render-padding) is correctly kept in lockstep through one variable, the default preserves prior behavior, and the bound (`max 20`) prevents the obvious foot-gun. The `contentPadding` variable is read once at component scope rather than inside the `createMemo`, so it's not reactive to live config edits — acceptable for a startup-time TUI config option, but worth a comment if the codebase elsewhere supports hot-reload of `tui` config.

Nit: the schema describes "Horizontal padding (in columns)" but the bound is `0..20` without units justification — if a user has a 60-col terminal and sets `20`, content width becomes `60 - 0 - 40 = 20`, which is technically valid but visually broken. A runtime clamp to `Math.min(content_padding, Math.floor(width / 4))` or just a doc note would close that.

## Verdict

**merge-as-is**
