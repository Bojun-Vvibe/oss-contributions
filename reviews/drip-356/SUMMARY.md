# drip-356 — 2026-05-05

## Verdicts

`(1, 5, 1, 1)` = (merge-as-is, merge-after-nits, request-changes, needs-discussion)

## Carrier coverage

4 of 7 carriers (sst/opencode ×3, openai/codex ×3, BerriAI/litellm,
google-gemini/gemini-cli). QwenLM/qwen-code, block/goose, and
charmbracelet/crush had no fresh (not-yet-reviewed) PRs in their
most-recent open-PR windows — every candidate was already in
INDEX.md from prior drips (qwen 3814–3840, goose 8994–9006,
crush 2773–2791 all DUP). Doubled up on the four active carriers
per the carrier-rotation rule.

## PRs reviewed

- `sst/opencode#25798@ad9eefd48719d80e6c6a2b80764ee3e5f2994035` — merge-after-nits
- `sst/opencode#25797@047fdd65f296672937cc03f82f3994b8c8434002` — merge-as-is
- `sst/opencode#25762@4c7cf5639030b394337f21ded6afecea7c84ce3d` — request-changes
- `openai/codex#21124@2558aafa2a1903fbbf0a8c92706ae83affd8e0c8` — merge-after-nits
- `openai/codex#21110@329222a4a73a60fee9560b46394c6cd8787214a5` — needs-discussion
- `openai/codex#21092@a0124597d7353b5ec5e886b0c1cfc2a7ea85fbc2` — merge-after-nits
- `BerriAI/litellm#27147@a2776fe7cadfe18090199a7e239d9bd1284557a5` — merge-after-nits
- `google-gemini/gemini-cli#26479@a7f309adb46349df98b97eafcf1e54102a710072` — merge-after-nits

## Cross-PR observations

**Theme: "right intent, weak guard."** Three independent PRs in this
batch ship a real bug fix wrapped in a guard whose semantics are
under-specified.

- opencode #25762 ships a five-pattern regex denylist for
  `kill node`-style commands — easily bypassed by `kill -9 $(pgrep
  node)`, `taskkill /F /PID <agent_pid>`, or any `Stop-Process -Name
  'node'` variant. The prompt-level warning is the actual defense;
  the regex creates a false sense of safety.
- gemini-cli #26479 adds `schedulerId` to every `TOOL_CALLS_UPDATE`
  test fixture but the back-compat semantic for legacy publishers
  that omit it isn't documented in the diff.
- codex #21110 introduces `largeContent: "deferred"` as a new
  discriminant in ~10 existing v2 response/notification types. Any
  ACP / TUI / mobile client that pattern-matches on the content
  variant will silently render empty image cells until updated. No
  server capability flag visible in the diff.

**Theme: protocol-additive PRs are landing cleanly.** codex #21124
(plugin share ACLs), codex #21092 (cwd/profile on experimental
feature list), opencode #25797 (nullable workspace warp id) — all
three are well-shaped additive schema changes with the right
nullable-by-default discipline. The minor inconsistency (codex
#21124 mixes SCREAMING_CASE `LISTED`/`UNLISTED`/`PRIVATE` with
snake-case `user`/`group`/`workspace` in two enums introduced in
the same diff) is the only stylistic flag.

**Theme: build-time generation is winning over checked-in derived
artifacts.** litellm #27147 deletes a 40k-line vendored backup JSON
in favor of `RUN cp` at wheel/Docker build time. The mechanical
shape is right; the `pyproject.toml`-as-witness check in
`get_model_cost_map.py:38-58` is fragile for vendored/monorepo
consumers and the `RUN cp ...` line is duplicated in 5 Dockerfiles
that should share a `prepare_build_artifacts.sh`.

## Standout findings

- **No security-bearing PRs in this batch.** No new credential
  literals, no auth-bypass patterns, no plaintext-secret regressions.
- **No regressions flagged for already-merged work.**
- **opencode #25762 is the only one I'd actively block** — not for
  what it does (the bug class is real) but for the combination of
  a trivially-bypassable regex with no tests and prompt-text
  duplicated across two files that will silently drift.

## Scrub actions

None. No PR diffs in this batch contained literal credentials, OAuth
client_secret values, API keys, or `.env`-style content. The
litellm #27147 diff is mostly Dockerfile and a deleted 40k-line
JSON of model pricing metadata — no secret material. Codex
#21124's plugin share ACL schema introduces `principalId` as a
field name but no concrete IDs appear in the diff. All review
files use `<placeholder>` style language only conceptually; no
literal secret text was written into any review file.
