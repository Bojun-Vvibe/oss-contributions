# Review: sst/opencode PR #25540

- **Title:** feat(tui): add logo.animate and logo.sound config options
- **Author:** K-log (Noah Dommaschk Burwell)
- **Head SHA:** `a6a2767dabbd77ec5d3d3ba715c045199add3526`
- **Verdict:** merge-after-nits

## Summary

Adds opt-in TUI config flags `logo.animate` and `logo.sound`, both
defaulting to `false`. The `Logo` and `GoLogo` components now gate
their animation `setInterval(tick, 16)` start, mouse-press handlers,
and all `Sound.start/pulse/dispose` calls behind those props. Schema,
migration, tests, and docs updated together. Closes #22528.

## Specific-line comments

- `packages/opencode/src/cli/cmd/tui/config/tui-schema.ts:27-39` —
  schema is well-structured: nested `logo` object with optional
  booleans, descriptions on every leaf. The "Only has effect when
  animate is true" callout on `sound` is a small but real UX assist
  for the JSON schema consumers (IDE tooltips).
- `packages/opencode/src/cli/cmd/tui/component/logo.tsx:582,604,627,638,663`
  — animation/sound gating is consistent across all five entry points
  (`tick`'s sound branch, `start`, `press`, `burst`, and the pulse on
  release). The `if (props.animation !== true) return` early-out in
  `start` is the right place to short-circuit; the `setInterval` never
  starts so there's no cleanup concern.
- `packages/opencode/src/cli/cmd/tui/component/logo.tsx:610` —
  `Sound.dispose()` only fires when `props.sound === true`. This is
  fine because `Sound.start/pulse` are also gated, but `Sound.dispose`
  is idempotent / safe to always call; gating it is defensive but
  adds a small footgun if someone toggles `sound` at runtime (cleanup
  would be skipped on the prior render). Toggles aren't a supported
  flow today, so OK.
- `packages/opencode/src/cli/cmd/tui/config/tui-migrate.ts:24,94` —
  legacy migration plumbs `logo` through `TuiLegacy` and the
  short-circuit guard. Good — without this, an existing `opencode.json`
  with a `tui.logo` block would not be migrated to `tui.json`.
- `packages/opencode/test/config/tui.test.ts:631-678` — three tests
  cover load-from-tui.json, undefined-when-absent, and legacy-migration
  including round-trip read of the migrated `tui.json`. Coverage is
  appropriate.

## Risks / nits

- Behavior change: animation+sound were previously always on; this PR
  flips both defaults to `false`. The PR description acknowledges
  this. Worth a one-line entry in the changelog/release notes —
  existing users will notice the home screen got quieter.
- Test names in `tui.test.ts` say `animate_logo` / `logo_sound` but the
  schema fields are `logo.animate` / `logo.sound`. Cosmetic — rename
  the test labels to match the actual field paths.

## Verdict justification

Clean opt-in flag with consistent gating, schema descriptions,
migration support, and tests. The default-off behavior change is the
only thing that needs a heads-up note. **merge-after-nits.**
