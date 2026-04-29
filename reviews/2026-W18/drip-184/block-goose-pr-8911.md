---
pr: block/goose#8911
sha: f70136bc9dd1a7d71131f7a9867ac1d2fc80f934
verdict: needs-discussion
reviewed_at: 2026-04-30T00:00:00Z
---

# goose2 distribution bundling

URL: https://github.com/block/goose/pull/8911
Files: `crates/goose/src/config/base.rs`, `ui/goose2/AGENTS.md`,
`ui/goose2/distro/README.md` (new), `ui/goose2/distro/config.yaml` (new,
empty), `ui/goose2/distro/distro.json` (new),
`ui/goose2/src-tauri/src/commands/distro.rs` (new),
`ui/goose2/src-tauri/src/commands/mod.rs`, `ui/goose2/justfile`, plus
frontend wiring
Diff: 595+/12-

## Context

Introduces a "distro bundle" packaging concept for the goose2 Tauri
shell — a discovery-resolved directory (`GOOSE_DISTRO_DIR` env override
or bundled Tauri resource at `resource_dir()/distro`) that ships
optional `distro.json` (manifest with feature toggles,
`providerAllowlist`, `extensionAllowlist`, `appVersion`),
`config.yaml` (Goose config layered into `goose serve`), and `bin/`
(executables prepended to `PATH` for the spawned `goose serve`). The
frontend reads the manifest at startup, stuffs it into Zustand, and
uses it to filter provider lists in Settings, filter the chat model
picker, and hide cost UI when `featureToggles.costTracking === false`.

## What's good

- Backend hook in `crates/goose/src/config/base.rs:164-168,179` is the
  cleanest part: a new `additional_config_paths_from_env()` helper
  reads `GOOSE_ADDITIONAL_CONFIG_FILES`, splits via `env::split_paths`
  (so colon-on-Unix / semicolon-on-Windows just works), and the
  `Config::default` impl extends `config_paths` with the result before
  appending `user_config_path`. Layering order matters: bundled
  defaults < distro extras < user config — that's the right precedence
  for "distro suggests, user wins."
- `distro/README.md` is unusually good for a new-feature doc:
  explicitly calls out that `providerAllowlist` and `extensionAllowlist`
  are **UI suggestions only** ("They do not enforce backend access
  control and do not invalidate existing sessions or saved model
  choices") — that disclaimer is the load-bearing security/UX caveat
  for anyone tempted to treat the manifest as a permissions system.
- Discovery resolution in `ui/goose2/src-tauri/src/commands/distro.rs`
  is documented with the right precedence: `GOOSE_DISTRO_DIR` first
  (development override), bundled Tauri resource second (production),
  no fallback — that means a packaged build with no distro silently
  works (frontend just gets `null`), which is the right "feature is
  optional" behaviour.
- `justfile:90-114, 146-170` automatic dev-mode export of
  `GOOSE_DISTRO_DIR` to `${PROJECT_DIR}/distro` when the directory
  exists, gated on `[[ -z "${GOOSE_DISTRO_DIR:-}" && -d "${DISTRO_DIR}"
  ]]` so an explicit operator override wins. The `echo "Using distro
  dir: ..."` at `:114, :170` makes the active distro visible in the
  console — useful for debugging "why is provider X hidden?"
- `distro/distro.json` ships with `featureToggles.costTracking: true`
  (default-on for the dev-shipped distro) — confirms the
  "omitted == enabled" contract documented in the README rather than
  silently shipping cost-tracking off in dev builds.
- `AGENTS.md:133` cross-link to `distro/README.md` is the right
  documentation hygiene move — agents inspecting the goose2 tree get
  pointed at the canonical doc instead of inferring intent from JSON
  files.

## Discussion items (why needs-discussion, not merge-after-nits)

1. **`providerAllowlist` is UI-only by design — but is that the right
   choice?** The README disclaimer is correct that the manifest doesn't
   enforce backend access. So a "distro" packaged for a regulated
   environment (the README's "Good fits" section explicitly mentions
   "packaged-app policy") will hide non-allowlisted providers in the
   picker, but: (a) any saved chat session pinned to a now-hidden
   provider continues to work, (b) the user can always type the model
   name into a free-form provider field if one exists, (c) `goose
   serve` itself doesn't know about the allowlist, so direct ACP
   clients (CLI, third-party) bypass entirely. If the intent is
   "policy", this needs an enforcement layer too; if the intent is
   "default UI affordance", call that out more loudly so distros aren't
   shipped with a false expectation of containment.
2. **`GOOSE_ADDITIONAL_CONFIG_FILES` is a new global env var on the
   `Config::default()` path** — that means it affects every consumer of
   the `goose` crate, not just goose2-launched `goose serve`. Is that
   intended? A standalone `goose` CLI invocation in a shell that
   inherited the env var will silently pick up a distro config it knows
   nothing about. Consider naming it `GOOSE2_ADDITIONAL_CONFIG_FILES`
   or scoping it via a new `Config::default_with_extras(extras: &[...])`
   constructor that goose2 calls explicitly.
3. **The empty `config.yaml` at `ui/goose2/distro/config.yaml`** —
   shipping a zero-byte file is fine but invites the question of
   whether distro authors are expected to populate it with the entire
   `goose serve` config (in which case it conflicts with user config)
   or only deltas (in which case the merge order needs to be
   contractually defined and tested). The README says "optional Goose
   config passed to `goose serve`" without specifying merge semantics.
4. **`featureToggles` typing is `Record<string, boolean>` — open-ended,
   only one toggle defined (`costTracking`).** Reasonable for a v0,
   but worth deciding now whether unknown toggles should be (a) silently
   ignored (current implicit behaviour), (b) logged at startup for
   forward-compat warnings, or (c) rejected at parse time to catch
   typos. Pick one before distros in the wild start depending on the
   "silent" semantics.
5. **No test in the diff** — neither the Rust `additional_config_paths_from_env`
   helper nor the frontend distro-manifest reader appears to have a
   unit test. For a feature whose bug class is "config silently
   doesn't apply," at minimum add (a) a Rust test that
   `GOOSE_ADDITIONAL_CONFIG_FILES=/tmp/a.yaml:/tmp/b.yaml` produces
   the right `config_paths` order, and (b) a frontend test that a
   missing/null manifest doesn't crash provider-list filtering.
6. **Backwards compatibility** — does an older `goose` binary
   without `GOOSE_ADDITIONAL_CONFIG_FILES` support gracefully degrade
   when launched by a goose2 shell that sets the env var? It would
   (the env var is just ignored), but worth confirming the
   `min_goose_version` or compatibility doc is updated.

## Verdict reasoning

The feature is well-motivated and the README is unusually good about
naming the UI-only-vs-policy boundary. The backend hook is the right
shape. But the env-var naming is too generic for a goose-wide global,
the "UI suggestion vs policy enforcement" distinction needs to be
either accepted-as-is with louder caveats or paired with a real
enforcement layer, the `config.yaml` merge semantics are
under-specified, and there are no tests for either the env-var loader
or the manifest parser. These are design discussion items, not
mechanical nits — worth resolving before this lands and downstream
distros start depending on emergent behaviour.
