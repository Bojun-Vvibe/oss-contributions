# Review: block/goose #8911

- **Title:** goose2 distribution bundling
- **Head SHA:** `e68c8026f346311b2a707db3d79d4c1d8d9b868a`
- **Scope:** +592 / -18 across 25 files
- **Drip:** drip-339

## What changed

Introduces a "distro bundle" concept for the goose2 Tauri UI: the desktop
app can now ship with a curated set of providers/configs baked in, and
the UI surfaces those constraints during provider selection. New
`distro_bundle.rs` service (+165), new ACP serve helper (+64), provider
constraint hook on the React side, and a `distroStore.ts` (+18) Zustand
store. Also bumps `tauri.conf.json` (+3) and adds a justfile target
(+16).

## Specific observations

- `ui/goose2/src-tauri/src/services/distro_bundle.rs` (+165 / -0): new
  service is the heart of the change. Look for filesystem reads of the
  baked-in `distro.json` — make sure the path is resolved via Tauri's
  `path_resolver().resolve_resource()` rather than a relative `./`
  lookup, otherwise it breaks for installed (non-dev) bundles.
- `ui/goose2/src-tauri/src/services/acp/goose_serve.rs` (+64 / -0): new
  helper to serve the bundled config over ACP. Confirm it errors loudly
  (not silently) when the distro JSON is malformed; the
  `useAppStartup.ts` hook (+33 / -1) appears to gate the entire UI on
  this call.
- `ui/goose2/src/features/providers/distroProviderConstraints.ts`
  (+34 / -0) plus its test (+63 / -0): nice to see the constraint logic
  isolated and unit-tested. The test file is the same size as the
  implementation — good ratio.
- `ui/goose2/src-tauri/tauri.conf.json` (+3): three lines added,
  presumably the `bundle.resources` entry for `distro.json`. Verify the
  glob pattern actually picks up `ui/goose2/distro/distro.json` on all
  three target OSes (macOS bundle resources are case-sensitive).
- `crates/goose/src/config/base.rs` (+7 / -0): touching the shared Rust
  config crate from a goose2-UI-only PR is a small but real coupling.
  Confirm no other crate depends on the modified surface in a way that
  would reject `None`/legacy configs.
- `ui/goose2/distro/config.yaml` (+0 / -0): zero-byte placeholder —
  intentional? If yes, add a `.gitkeep`-style comment so it isn't
  flagged as "added empty file" by reviewers.

## Risks

- Resource path resolution differing between `tauri dev` and the
  packaged app is the classic foot-gun here.
- Coupling the UI startup hook to a successful distro fetch can brick
  the app if the bundle is missing/corrupt.
- `crates/goose/src/config/base.rs` change is the only non-UI surface
  change; needs explicit cross-crate test confirmation.

## Verdict

**merge-after-nits** — verify Tauri resource resolution, add a
malformed-distro fail-closed test, and either justify or split the
`crates/goose` change.
