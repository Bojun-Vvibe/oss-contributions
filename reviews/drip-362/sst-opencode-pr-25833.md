# sst/opencode #25833 — fix(app): respect safe area in web shell

- **Head SHA:** `94aa4538cccb172085dcced20826e0cba2821d76`
- **Author:** @Hona
- **Verdict:** `merge-after-nits`
- **Files:** +8 / −1 across `packages/app/index.html`, `packages/app/src/index.css`

## Rationale

Two-line fix that adds `viewport-fit=cover` to the meta viewport at `packages/app/index.html:5` and applies `env(safe-area-inset-*)` padding to `#root` in `packages/app/src/index.css:3-8`. This is the standard recipe for letting a PWA paint into the iOS notch/home-indicator gutters without having content occluded — without `viewport-fit=cover`, mobile Safari ignores `env(safe-area-inset-*)` entirely, so both pieces of this diff are required and correctly co-landed.

The implementation choices are mostly defensible. Pinning `#root` to `100vh` then re-declaring `100dvh` (lines 3-4) correctly handles older WebKit that doesn't grok dynamic viewport units; the second declaration silently wins on newer browsers via the cascade. Using `max(1px, env(...))` is the standard guard against zero-padding on non-iOS where the env var is `0px` — though I'd argue `1px` is a slightly weird floor (it gives every shell a 1px sliver of inset for no obvious reason); a more honest pattern would be `env(safe-area-inset-top, 0px)` directly, since iOS reports a real value when the notch matters and `0px` everywhere else is exactly what we want. **Nit:** if the `1px` floor is intentional (e.g. to give the root element a hairline gutter so children with `position: absolute; top: 0` don't kiss the viewport edge), please drop a comment explaining why; otherwise drop it.

The other thing worth flagging: `#root { height: 100vh }` is a global side-effect on a single CSS selector that's typically owned by the framework's own root — if `@opencode-ai/ui/styles/tailwind` (imported on line 1 of the same file) ever ships its own `#root` rule, source order means this one wins, but a future tailwind preset bump could silently start fighting it. Scoping with a body class (e.g. `body[data-shell="web"] #root { ... }`) would be safer for the small amount of effort. None of these are blockers — the PR fixes a real, observable bug on iOS PWA installs and the diff is 8 lines. Land after a quick decision on the `1px` floor.
