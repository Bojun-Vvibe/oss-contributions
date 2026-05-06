# Review: QwenLM/qwen-code #3863 — feat(cli): add Anthropic model listing support (Option A)

- Carrier: `QwenLM/qwen-code`
- PR: #3863
- Head SHA: `fa361456`
- Drip: 387

## Verdict
`request-changes` (rc)

## Summary
The intent (add Anthropic `/v1/models` listing support to the qwen-code
provider registry, "Option A" branch of a previously-discussed design)
is reasonable, but **this PR is not in a mergeable state**. The diff
includes substantial unrelated content that must be removed before any
review of the actual feature can land.

## Blocking issues

### 1. Committed binary artifacts
Three SQLite databases committed to `.gemini_security/`:
- `.gemini_security/graphiti.db` (binary)
- `.gemini_security/pulse.db` (binary)
- `.gemini_security/second_brain.db` (binary)

These are local agent-tooling state from the contributor's machine. They
have no business in the source tree:
- They will silently leak whatever the contributor's local agent
  stack indexed (file paths, possibly content snippets, possibly
  credentials referenced during indexing).
- They make every future PR's `git status` show stale-binary noise.
- They cannot be reviewed (binary diff).

**Action:** strip from history before merge; add `/.gemini_security/` to
`.gitignore`.

### 2. Committed contributor IDE configuration
`.serena/.gitignore` and `.serena/project.yml` (119 lines) are
Serena-specific local IDE config naming the project, declaring the
language, configuring read-only modes, etc. This is per-developer state
and belongs in the contributor's untracked workspace, not the repo.

**Action:** strip; add `/.serena/` to `.gitignore`.

### 3. Committed agent session log
`RUN2.md` (599 lines) is a verbatim transcript of the contributor's
local Claude Code session — including hook-error messages naming
absolute paths under `/home/bamn/.claude/hooks/`, a username, command
outputs, partial PR-review automation traces, and several `gh api`
responses with PR comment IDs. This is operational noise and a privacy
leak.

**Action:** strip entirely. Session transcripts never belong in
source control.

### 4. Diff scope mismatch
The PR title is "feat(cli): add Anthropic model listing support
(Option A)" but the actually-feature-touching code is invisible in
the diff prefix — it's buried under hundreds of lines of the above
artifacts. A reviewer cannot tell what the feature implementation
looks like without scrolling past 700+ lines of unrelated noise.

**Action:** rebase or `git reset --soft` and re-commit only the
feature changes.

## What I can confirm without scrolling further
- The PR's stated intent (Anthropic `/v1/models` listing as "Option A"
  of a prior design discussion) maps to existing patterns in
  `packages/cli/src/config/auth/` based on prior model-list PRs in
  this repo.
- The contributor (`B-A-M-N`, visible in the embedded session log) has
  a working local automation pipeline ("PRForge skill", review IDs
  4215773687 / 4215939438 / 4215940090) that suggests they're
  iterating on review feedback — but the iteration has happened in
  *content* the maintainer must not see.

## Recommendation
- **Block** on the four issues above.
- Once the contributor force-pushes a clean branch containing only
  the Anthropic model-listing changes, this becomes a normal
  feature-review PR and a maintainer can verify Option A's design
  (likely: a new `listAnthropicModels()` call wired into the
  provider registry, with caching keyed on the API key fingerprint).
- The contributor should also be advised to add `/.gemini_security/`,
  `/.serena/`, and any `RUN*.md` glob to their global gitignore so
  this doesn't recur.

## Concerns flagged for the next review pass (after cleanup)
- **Anthropic's `/v1/models` is paginated.** Any client implementation
  must follow `next_page` cursors or it will silently return only the
  first page (~20 models).
- **API-key-fingerprint caching.** A naive cache keyed on the raw API
  key would land the key in process memory; should use a
  SHA-256 prefix or the existing fingerprint helper used elsewhere
  in this repo.
- **Anthropic model IDs include version dates** (e.g., `claude-3-5-
  sonnet-20240620`). The provider's existing model-name resolution
  treats these as opaque strings, but downstream `model_router`
  logic that does prefix matching may need updating.

## Risk
Cannot be assessed until cleanup is complete. As-shipped, this PR
would commit binary blobs and local IDE state to the public history.
