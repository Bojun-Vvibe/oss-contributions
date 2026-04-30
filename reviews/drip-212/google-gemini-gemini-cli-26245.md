# google-gemini/gemini-cli PR #26245 — Changelog for v0.40.0

- Repo: `google-gemini/gemini-cli`
- PR: https://github.com/google-gemini/gemini-cli/pull/26245
- Head SHA: `310398bba6fec115383d7bec3826e74469b44ef7`
- State: OPEN
- Files: 2 (`docs/changelogs/index.md`, `docs/changelogs/latest.md`), ~+150/−~155
- Verdict: **merge-as-is**

## What it does

Pure documentation: bumps the "latest stable" pointer from `v0.39.0`
(2026-04-23) to `v0.40.0` (2026-04-28) and prepends a v0.40.0
"Announcements" block to `docs/changelogs/index.md`. Body of
`docs/changelogs/latest.md` is replaced wholesale with the v0.40.0
release notes (fresh "Highlights" + "What's Changed" + "Full Changelog
compare URL"). All linked PRs are real and resolve to merged
contributions in the v0.39.1...v0.40.0 range.

## What's right

- This is a `gemini-cli-robot` automated changelog PR. Right
  boundary: docs only, no code paths touched. Mechanical content
  generated from merged-PR titles + authors.
- Both the `index.md` "Announcements" block and the `latest.md`
  rewrite share the same 6 highlights with consistent wording and
  cross-link to the same 6 anchor PRs (`#25342, #15504, #25395,
  #25716, #25586, #25498`). The two surfaces don't drift.
- Highlights are honestly grouped:
  - **Offline Search** (`#25342`) — bundled `ripgrep` into the SEA
    binary
  - **Themes** (`#15504`) — colorblind theme variants
  - **MCP Resource tooling** (`#25395`) — list/read MCP resources
  - **Memory tier refactor** (`#25716`) — `MemoryManagerAgent`
    replaced with prompt-driven memory editing across 4 tiers
  - **Topic narration** (`#25586`) — enabled by default
  - **`gemini gemma` setup** (`#25498`) — local model bootstrap
- The "Full Changelog" footer links to
  `https://github.com/google-gemini/gemini-cli/compare/v0.39.1...v0.40.0`,
  which is the right base (v0.39.1 was the last stable, not v0.39.0).

## Notes

- The two cherry-pick PRs at the tail (`#25942` and `#26124`) are the
  preview-branch patches that landed on top of v0.40.0-preview.3 and
  v0.40.0-preview.4 respectively; they're listed in the "What's
  Changed" body, which is correct since the v0.40.0 stable
  necessarily contains both.
- `#26124` is annotated `[CONFLICTS]` in the listing — that's the
  upstream cherry-pick robot's literal title. Not a docs bug; the PR
  title itself contains the marker because the cherry-pick was
  hand-resolved.
- The previous changelog (v0.39.0) had an "Enhanced Plan Mode
  Security" highlight that explicitly named user-confirmation for
  skill activation — v0.40.0's highlights don't reference any
  Plan Mode follow-up, but the body lists `#25515 reset plan session
  state on /clear` and `#25317 silent fallback for Plan Mode model
  routing` as Plan Mode work. Reasonable that those didn't make
  Highlights, but a single Plan Mode bullet would be a fair
  representation. Not blocking — Highlights are editorial.

## Nits (none blocking)

1. The `#25942` and `#26124` cherry-pick lines are functionally
   noise to end users (they're internal release-branch mechanics,
   not user-facing changes). Some upstream changelogs filter
   `cherry-pick` PRs out of "What's Changed". Worth a future
   convention discussion, but not for this PR.
2. The blank line around "Reduce blank lines." (`#25563` by
   @gundermanc) is a minor source-readability win the new format
   could acknowledge with one less blank line. Editorial.

## Why merge-as-is

Auto-generated changelog from a release-bot, content matches the
underlying PRs, both surfaces (index + latest) stay in sync, and the
v0.39.1 base for the compare URL is correctly chosen. No code paths,
no API contracts, no migration notes needed. Merge.
