# sst/opencode#25198 — fix: fix AI refusing to commit

- **PR**: https://github.com/sst/opencode/pull/25198
- **Author**: scarf005
- **Head SHA**: `dbf6fc674349b1c7e70e8d4862dd1a55631bf188`
- **Files**: `packages/opencode/src/session/prompt/default.txt`, `packages/opencode/src/session/prompt/trinity.txt`, `packages/opencode/src/tool/bash.txt`
- **Verdict**: **needs-discussion**

## Context

User report: the agent refused to commit when *explicitly asked* to commit, citing the system-prompt rule `"NEVER commit changes unless the user explicitly asks you to."` The PR removes that line from `default.txt`, `trinity.txt`, and `bash.txt`, plus rewrites the bash-tool committing-changes preamble from `"Only create commits when requested by the user. If unclear, ask first. When the user asks you to create a new git commit..."` to `"When creating a new git commit, follow these steps carefully:"`.

## What's right

- **The reported failure mode is real.** Some models (notably Claude variants tuned strongly on Anthropic's instruction-following datasets) treat *any* repetition of a "NEVER do X" rule as load-bearing even when X has just been explicitly requested. Removing the duplicate copies of the same rule reduces the chance of over-steering.
- **The duplicate is gone in three places.** `default.txt:86`, `trinity.txt:78`, and `bash.txt:66` all carried the same line. Keeping it in one canonical place (the bash-tool committing-changes block) and removing the other two is the right deduplication direction.

## Risks / nits

- **The bash-tool preamble rewrite drops the explicit-ask gate entirely**, not just the duplicate sentence. Pre-fix `bash.txt:53` read `"Only create commits when requested by the user. If unclear, ask first."` Post-fix it reads `"When creating a new git commit, follow these steps carefully:"` — the "only when requested" + "if unclear, ask first" guidance is gone from the canonical location, not just deduplicated. The remaining safety bullets at `:55-65` are about *how* to commit (don't update git config, don't force push, don't `--amend` unless conditions met) — none of them say *when* to commit.
- **This is the load-bearing concern.** A user asking "fix this bug" no longer has any prompt-level guard against the agent committing on its own. The original rule existed because models will proactively commit "to save your work" when not asked, and that's a real footgun in workflows where the user wants to review the diff before committing. The fix-the-symptom direction here is "remove the rule entirely" rather than "fix the rule's wording so it doesn't fire on explicit-ask cases" (e.g. `"Only commit when the user has explicitly asked you to in the current turn or via project AGENTS.md"`).
- **No before/after eval evidence in the PR description.** A claim "AI refuses to commit" without a reproducible prompt + the model that exhibited the refusal makes it impossible to verify the removal actually fixes the reported issue vs. some other contributor (e.g. tool wiring, permission system, the `git_commit` tool's own consent flow).
- **The `default.txt` removal also takes out a blank line at `:87`** and leaves the `<system-reminder>` bullet adjacent to the lint/typecheck bullet without a separator — a cosmetic nit but the diff shows the structure shift.

## Verdict

**needs-discussion.** The reported symptom is plausible but the chosen remediation removes a guard that has different semantics from the duplicated sentence. Recommend either (a) keep one canonical "Only commit when explicitly asked" rule in the most authoritative location (`bash.txt` committing block) and remove only the duplicates, OR (b) replace with a positively-framed rule (`"Commit when the user asks you to. Don't commit proactively."`) so the explicit-ask case isn't ambiguous. Also wants a repro prompt + model in the PR body so the fix is verifiable.
