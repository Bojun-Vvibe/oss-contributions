---
pr: https://github.com/QwenLM/qwen-code/pull/3783
head_sha: 04d266c5f6e7d90714563d7c0a0f3f31a9b4acdc
author: alex-musick
additions: 16
deletions: 0
files_touched: 1
---

# Scope

Adds a new `/model [model-name]` syntax to the in-session model command
so users can switch the active model immediately without going through
the interactive selector. Existing forms (`/model`, `/model --fast`,
`/model --fast [name]`) are untouched. Closes upstream issue #3410.
Single-file change, 16-line diff.

# Specific findings

- `packages/cli/src/ui/commands/modelCommand.ts:95-107` (head
  `04d266c`) — the new branch is correctly gated on three conditions:
  - `args !== ''` (some argument was provided),
  - `!args.startsWith('--fast')` (don't shadow the existing fast-model
    branch),
  - `context.executionMode === 'interactive'` (PR body explicitly states
    non-interactive mode is excluded; the launch flag `--model` covers
    that case).
  Order is correct. One subtle issue: a user typing `/model --fastfoo`
  (no space) would also be caught by `startsWith('--fast')` and *not*
  routed to the new branch — fine, that string isn't a valid flag
  either way. But `/model --fas` (typo) would fall through to the new
  branch and `setModel('--fas')` — flagging an unknown model name with
  no validation. Suggest also rejecting any arg starting with `--`.
- `packages/cli/src/ui/commands/modelCommand.ts:101` — `args.trim().split(' ')[0]`
  takes the first whitespace-delimited token. PR body justifies this as
  "avoids later syntax confusion and/or use of model names with spaces"
  — reasonable. Note the comment has a typo: `synatx` → `syntax`.
- `config.setModel(modelName)` at line 102 — there is no validation
  that the model name resolves against the current provider, no error
  handling if `setModel` throws, and the function returns a confident
  `Model: <modelName>` info message regardless. The PR body acknowledges
  this trade-off explicitly ("This feature allows the user to try
  switching to an unavailable model at runtime. This replicates the
  functionality of the launch argument `--model` and is intended.").
  Acceptable for parity with launch behavior, but consider returning a
  warning-flavored message ("Set model to X — verify availability with
  the next request") so users aren't misled into thinking it succeeded
  silently.
- No tests added. The existing `/model` interactive flow is presumably
  covered elsewhere; a unit test asserting `args="gpt-4o"` →
  `setModel("gpt-4o")` was called and the right info string was
  returned would be cheap and protect against future refactors of the
  argument-parsing branch order.
- The PR body claims `npm run build`, `npm run test:scripts`, and
  `npm run start` all pass; no CI status visible in the diff metadata.

# Suggested changes

1. Reject any `args` starting with `--` in the new branch (so `/model
   --fas` doesn't try to set the model to `--fas`); fall through to the
   existing usage / interactive selector instead.
2. Fix the inline comment typo `synatx` → `syntax`.
3. Add a unit test that exercises the new branch with a sample model
   name and asserts both `config.setModel` was called and the returned
   message has the expected shape.
4. Optional: soften the success message to indicate the change is
   unverified (e.g. "Active model set to X. Availability will be
   confirmed on next request.").

# Verdict

`merge-after-nits`

# Confidence

High — the change is small, correctly scoped, and matches the
documented launch-flag behavior; the remaining items are local polish.

# Time spent

~7 minutes.
