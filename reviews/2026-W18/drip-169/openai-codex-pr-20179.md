# openai/codex#20179 — `TUI: Remove core protocol dependency [7/7]`

- PR: https://github.com/openai/codex/pull/20179
- Head SHA: `4fc35a1271014414ee82899061c1f0f1f0b8c71b`
- Author: etraut-openai

## Summary of change

Closes the seven-part series that severs `codex-tui` from
`codex-core`. The final PR tightens the boundary-verification script
in `.github/scripts/verify_tui_core_boundary.py` so that, in addition
to forbidding `codex_core::` / `use codex_core` / `extern crate
codex_core` in TUI source files, it now also rejects
`codex_protocol::protocol` (both bare and inside a `use codex_protocol::{... protocol ...}` brace import). Header docstring and CI failure
message updated to "stay behind the app-server/core boundary."

## Specific observations

- `.github/scripts/verify_tui_core_boundary.py:14-32` — the rules
  table is now a tuple of `(message, patterns)`. Cleaner than the
  flat `FORBIDDEN_SOURCE_PATTERNS` it replaces, and lets the script
  attribute each violation to a category in the failure output. Good
  refactor.
- `verify_tui_core_boundary.py:30` — second pattern is
  `re.compile(r"\bcodex_protocol\s*::\s*\{[^}\n]*\bprotocol\b")`.
  The `[^}\n]` class will *not* match across a multi-line braced
  import like `use codex_protocol::{\n    protocol,\n    other,\n};`,
  which rustfmt produces routinely. So `cargo fmt` on a violating
  file can silently smuggle a forbidden import past CI. Either drop
  the `\n` from the negated class (use `[^}]` and switch to
  `re.DOTALL`) or scan the file as one string with `re.MULTILINE +
  re.DOTALL`.
- `verify_tui_core_boundary.py:79-94` — the rule loop now iterates
  per-line, and on a hit appends `f"{relative_path(path)}:{line_number} {message}"`. Two pattern groups will produce two distinct
  failure lines for the same source line if both match; that's fine
  for TUI-author signal but worth knowing for CI log volume.
- The doc string update on line 3 ("stays behind the
  app-server/core boundary") and the failure message on line 33
  ("temporary embedded startup gaps belong behind
  `codex_app_server_client::legacy_core`, and core protocol
  references should remain outside `codex-tui`") match the
  architectural target of the series and give the next contributor
  who trips this a clear pointer to the escape hatch. Good DX.
- No corresponding allowlist for `codex_protocol::` (without the
  `::protocol` tail). That is intentional — the rest of the
  `codex_protocol` namespace stays available — but it's worth a
  one-line comment in the script explaining *why* only the
  `::protocol` submodule is fenced off.

## Verdict

`merge-after-nits` — the multi-line braced-import gap in the second
regex needs a fix, otherwise rustfmt becomes an attack vector
against the boundary. Everything else here is clean follow-up to a
multi-PR architectural cut.
