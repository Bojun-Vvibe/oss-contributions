# block/goose PR #8949 — install plugins

- **Author:** jamadeo
- **Head SHA:** `bf21ae16afd4a0fcdb33dbace084b9b2c15abe97`
- **Base:** `main`
- **Size:** +440 / -1 (multi-file scaffolding)

## What it does

Adds a new `goose plugin install <git-url>` CLI subcommand and a
`goose::plugins` module that clones a git repo, parses a
`gemini-extension.json` manifest, copies it into a per-plugin install
directory, and imports any embedded skills. The PR description is honest
that this is *scaffolding*: only the gemini format and skill imports are
implemented in this round.

## Diff observations

- `crates/goose-cli/src/cli.rs:17,636-647,857-862,1066,1531-1535,1865` —
  registers a new `Plugin` subcommand with a single `Install { url }` arm,
  wires it into `get_command_name` for telemetry, adds the dispatcher
  `handle_plugin_subcommand`, and matches it in the top-level `cli()` arm.
- `crates/goose-cli/src/commands/plugin.rs` (new, 27 lines) — minimal
  presenter: green check, format, name, version, source, install
  directory, and an itemized list of imported skills.
- `crates/goose-cli/src/commands/mod.rs:5` — `pub mod plugin;`.
- `crates/goose/Cargo.toml:145` — drive-by switch `tempfile = {
  workspace = true }` → `tempfile.workspace = true` (cosmetic only).
- `crates/goose/src/lib.rs:30` — `pub mod plugins;`.
- `crates/goose/src/plugins/formats/gemini.rs` (new, 191 lines) — manifest
  parser. Key pieces:
  - `MANIFEST = "gemini-extension.json"`.
  - `try_install_from_manifest` returns the sentinel `FormatNotSupported`
    if the manifest file is absent (allowing the orchestrator to try other
    formats — anticipated future work).
  - Validates manifest `name` via `validate_extension_name` (not shown in
    this slice; presumably enforced in the parent module).
  - Refuses to overwrite an existing install with `bail!("Plugin '{}' is
    already installed at {}")`.

## Strengths

- Clear seam: format-specific code lives behind a
  `try_install_from_manifest(...)` that returns `FormatNotSupported`. Adding
  open-plugins / claude marketplace formats later means another file in
  `plugins/formats/`, not a fork of the dispatch logic.
- The CLI presenter is read-only and doesn't try to also handle the
  install side — separation of concerns is right.
- The PR body explicitly enumerates "what's not in this PR" (other
  formats, GUI install, updating). Good scoping.

## Concerns

- **Security**: this clones an *arbitrary* user-supplied git URL and copies
  files into a privileged install directory. The visible diff slice
  doesn't show:
  - URL allow-listing or scheme validation (file://, ssh://, etc.).
  - Branch/tag pinning (does it follow `HEAD`? Can a plugin author
    silently push new code that runs next time the user invokes the
    plugin?).
  - Path-traversal hardening of `manifest.name` (used as
    `install_root.join(&manifest.name)` — a name like `../foo` would
    escape `install_root`). `validate_extension_name` is called but its
    body isn't in this diff slice; please confirm it rejects `..`, `/`,
    `\`, and absolute paths.
  - Skill content sandboxing (skills can carry executable code on most
    agent stacks).
- **No tests** on the install path are visible in this diff slice. For a
  feature that downloads code from the internet and copies it into a
  user's profile, the bar for tests is high — at minimum a path-traversal
  test for `validate_extension_name`, a "manifest missing" test
  exercising `FormatNotSupported`, and a happy-path test using a fixture
  directory (no network needed if the git step is factored behind a
  trait).
- **Update story is missing.** `bail!("already installed")` is the right
  default but means there is no `goose plugin update` / `goose plugin
  reinstall`. This is acknowledged in the PR body; users will discover it
  fast.
- The CLI uses `console::style("✓").green()` directly. If goose has a
  shared output helper (it does in other commands), reuse it for theme
  consistency.

## Nits (non-blocking)

- `tempfile = { workspace = true }` → `tempfile.workspace = true` is a
  cosmetic Cargo.toml change unrelated to the feature; conventional to
  keep drive-bys out of the same commit, but harmless.
- The PR title `install plugins` is lowercase; the project's other recent
  PR titles use `feat(...): ...`. A `feat(cli): install plugins from git`
  title would help changelog tooling.

## Verdict

**needs-discussion** — feature shape and module boundaries look good, but
this is "download arbitrary code from a URL" plumbing and the diff slice
shown gives no answers on URL trust, manifest-name path-traversal
hardening, ref pinning, or test coverage. Want maintainer sign-off on the
threat model before this lands, even as scaffolding.
