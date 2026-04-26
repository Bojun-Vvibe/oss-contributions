# block/goose PR #8825 — chore(deps): bump lopdf from 0.36.0 to 0.40.0

- **PR:** https://github.com/block/goose/pull/8825
- **Author:** app/dependabot (bot)
- **Head SHA:** `f67b7cc35a3e2369a21140c0e349476104e31931`
- **Files:** 2 (+13 / -?)
- **Verdict:** `merge-after-nits`

## What it does

Dependabot bump of `lopdf` from 0.36.0 → 0.40.0 in `crates/goose-mcp`. lopdf is a pure-Rust PDF parser used by goose's MCP for reading PDF attachments. The bump skips three minor versions (0.37, 0.38, 0.39) and lands on 0.40.

## Specific reads

- `crates/goose-mcp/Cargo.toml:37` — single direct dep change:
  ```toml
  -lopdf = "0.36.0"
  +lopdf = "0.40.0"
  ```
  Caret semantics: this is `^0.40.0` so it accepts 0.40.x patch updates only — appropriate for pre-1.0 crate.
- `Cargo.lock:5872-5895` — lopdf entry updated with new checksum and dep changes:
  - `+ "getrandom 0.4.2"` added (used for new PDF decryption support per release notes)
  - `- "rand 0.9.2"` → `+ "rand 0.10.0"` (transitive bump alongside lopdf's own bump)
  - `+ "ttf-parser"` added — lopdf now depends on a TTF font parser, presumably for embedded font handling
- `Cargo.lock:7383-7388` — `itertools 0.13.0` → `0.14.0` for what looks like `proc-macro-error-attr2` (unrelated to lopdf, possibly a co-bump landing in the same lockfile regen).
- `Cargo.lock:11098-11104` — new `ttf-parser 0.25.1` crate enters the dependency graph.

## Risk surface

Moderate. Three concerns:

1. **Three minor versions skipped (0.37 → 0.38 → 0.39 → 0.40).** Per the release notes excerpt in the PR body, 0.39 added "pdf decryption support that derived from pdftk" — this is a non-trivial new code path. If goose's MCP ever calls into the decryption API on an attacker-supplied PDF, that's new attack surface. Worth confirming goose's PDF tool only invokes plain parse paths.
2. **New transitive deps.** `getrandom 0.4.2` (old major — getrandom is on 0.3.x as latest, so 0.4.2 here is suspicious — could be a parallel-major release), `rand 0.10.0`, and `ttf-parser 0.25.1` all enter the supply chain. None are individually scary, but a Dependabot bump that adds three new crates deserves a maintainer eyeball on the lockfile before approval — especially `getrandom 0.4.2` which doesn't match the latest stable `getrandom` line.
3. **No code changes in the goose-mcp PDF tool itself.** That's expected for a Dependabot bump, but it also means there's no test demonstrating the new lopdf still parses goose's existing test PDF fixtures. The goose PR CI should be running the MCP integration tests against the bumped crate; verify those passed before merge.

## Nits (not blocking)

1. Worth scanning `crates/goose-mcp/src/**` for any call into `lopdf::Document::*decrypt*` to confirm goose isn't accidentally exposing the new decryption surface to untrusted PDFs.
2. Consider pinning lopdf to `=0.40.0` (exact) until goose's eval suite has run against it for at least one release cycle. Pre-1.0 PDF parsing crates have a history of patch-level regressions.
3. The `itertools 0.13 → 0.14` co-bump should ideally be in a separate PR (it's not lopdf-driven), but not worth blocking.

Verdict: merge after a maintainer confirms (a) MCP PDF integration tests pass, (b) no `*decrypt*` API is exposed to untrusted input, and (c) the `getrandom 0.4.2` resolution is intentional and not a yanked/sidegraded version. Otherwise straightforward dep maintenance.
