# sst/opencode#24935 — `feat(tui): add terminal pet companion with CAVA audio visualizer`

- URL: https://github.com/sst/opencode/pull/24935
- Head SHA: `d74e8bf24b2f`
- Author: ZoeImport
- Size: +1448 / -0 across 10 files
- Closes: #24937

## Summary

Adds an ASCII pet to the TUI sidebar with a 6-state behavior
machine, weather influence, and an optional CAVA audio
visualizer. Ships under `examples/terminal-pet/` plus new feature
plugin files under `packages/opencode/src/cli/cmd/tui/feature-plugins/sidebar/`.
Two top-level design docs (`PR-UI-EXTENSION.md`, `PR_DESCRIPTION.md`)
are added at repo root.

## Specific references

- `PR-UI-EXTENSION.md` and `PR_DESCRIPTION.md` are added at the
  repository root. These are PR-meta files — they belong in the PR
  description box, not the tree. Likely to be rejected on style
  alone.
- `packages/opencode/src/cli/cmd/tui/feature-plugins/sidebar/pet.tsx`
  and `spectrum.tsx` are introduced as first-class TUI feature
  plugins. Per the PR's own analysis (`PR-UI-EXTENSION.md` lines
  ~10–25 listing existing slots), the project already exposes a
  `sidebar_content` plugin slot. A whole-cloth pet should live in
  an external `~/.config/opencode/plugins/...` package using that
  slot (which the PR even creates as `examples/terminal-pet/`),
  not be merged into the core TUI tree as a `feature-plugin`.
- CAVA dependency: `spectrum.tsx` shells out to `cava`. There's
  no detection / graceful-skip wired into the sidebar mount path
  visible in the diff — the PR description claims a fallback exists
  but the test surface is +0 lines, so nothing exercises it.
- `examples/terminal-pet/pets.config.json` and
  `pet.config.schema.json` add a config schema with no validator
  hooked up in the loader path.

## Risks

- Scope: this is two features (pet + audio visualizer) plus a
  proposed new "UI Widget Extension System" doc rolled into one
  1448-line PR. The maintainers' usual policy on this repo is one
  small change per PR.
- Maintenance burden: a stateful animation widget in the core TUI
  binary is forever the maintainer's problem. The
  `examples/terminal-pet/` external plugin form already
  demonstrates the same thing without taking on that burden.
- Zero tests added on a PR introducing a state machine, weather
  system, and audio integration.
- Top-level `.md` files at repo root will collide with future
  PR-template tooling.

## Verdict

`request-changes`

## Suggested direction

1. Drop the in-tree `feature-plugins/sidebar/pet.tsx` and
   `spectrum.tsx` additions; ship the whole thing as the external
   plugin under `examples/terminal-pet/` (already in the PR).
2. Move the two `PR-*.md` files into the PR description / a single
   `examples/terminal-pet/README.md`.
3. Hard-gate the CAVA path on a runtime `which cava` check with a
   silent no-op fallback, plus one snapshot test asserting the
   widget renders without `cava` present.
4. If the maintainers do want pet-as-core, split into three PRs:
   plugin-slot exposure (if any new slot is needed), pet widget,
   audio visualizer.
