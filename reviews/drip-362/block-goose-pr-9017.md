# block/goose #9017 — soften goose2 text selection polish

- **Head SHA:** `b19db05c0ea80b72a463f22adfb686f35344a173`
- **Author:** @morgmart
- **Verdict:** `merge-after-nits`
- **Files:** +16 / −3 across `ui/goose2/src/shared/styles/globals.css`, `ui/goose2/src/shared/ui/input.tsx`

## Rationale

Three independent adjustments bundled as "selection polish":

1. **Global `::selection` recipe softened** at `ui/goose2/src/shared/styles/globals.css:747-749`: the brand-color overlay drops from 60% to 18% opacity, and `color: var(--brand-foreground)` becomes `color: inherit`. This is the right call — at 60% alpha the previous overlay was effectively opaque on most backgrounds, which produced a "selected text becomes unreadable on dark mode" failure mode. 18% is mild enough to remain legible across light/dark while still being clearly distinct from un-selected text. `color: inherit` is also correct: the previous `var(--brand-foreground)` assumed the *user* selected text on the brand background, but selection happens on whatever background the text lives on — inheriting the existing color is the only choice that's right in all themes.

2. **`user-select` rule pair added** at lines 752-765: `button`, `[role="button"]`, and `[data-tauri-drag-region]` get `user-select: none`; `input`, `textarea`, `[contenteditable="true"]`, and `[data-streamdown]` get `user-select: text`. The `data-tauri-drag-region` exclusion is specifically for the desktop title-bar drag affordance — without it, drag-to-move on the window chrome would accidentally select adjacent text mid-drag, which is exactly the polish bug this PR's title alludes to. The opt-back-in for `[data-streamdown]` is the interesting one: streaming markdown chunks ship with that attribute and would otherwise inherit `user-select: none` from any ancestor button-like wrapper, breaking copy-from-LLM-response. Good catch.

3. **Input variant declass** at `ui/goose2/src/shared/ui/input.tsx:6`: removes `selection:bg-primary selection:text-primary-foreground` from the default input variant. This is the necessary cleanup — without removing the per-input `selection:` overrides, the new global `::selection` rule from change (1) would be silently shadowed inside every default input by the old hard-coded primary-color recipe, and you'd see inconsistent selection rendering between inputs and surrounding prose. Removing the local override lets the global rule take effect uniformly.

**Nits:**

1. **The `[data-tauri-drag-region]` selector embeds desktop-shell knowledge in a "shared/styles" file.** That coupling already exists in the codebase (it's the established convention for Tauri apps) so this isn't really a new sin, but worth a one-line comment in the CSS block at line 758 explaining *why* drag regions are in the no-select group — six months from now nobody will remember.
2. **`[data-streamdown]` is similarly opaque.** A brief comment ("streamed markdown content; allow text selection so users can copy LLM responses") would future-proof against someone tidying it up.
3. **No test, no Storybook screenshot.** This is pure visual polish so a unit test isn't really feasible, but a short before/after note in the PR body (or a Chromatic snapshot if the project uses one) would help the next person who wonders why the selection alpha is specifically 18%.
4. **Verify the i18n / RTL story.** The `user-select` rules are LTR/RTL-agnostic, but worth a quick eyeball that no RTL-specific input components rely on the now-removed `selection:bg-primary` for visibility.

Net: the cumulative effect is correct and the diff is small enough to review confidently. Land after the two clarifying CSS comments.
