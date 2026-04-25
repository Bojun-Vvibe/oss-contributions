# cline/cline#10382 — fix(hooks): Use shell escapes on JSON literals in hooks templates

- **URL**: https://github.com/cline/cline/pull/10382
- **Author**: dominiccooney
- **Head SHA**: `c925a85cffb5a0e36f02d0e27d1e2e8b2a347a13`
- **Verdict**: `merge-after-nits`

## Summary

This PR has a misleading title: it actually does two things at once.
First, it converts the JSON literal output in every hook template from
double-quoted shell strings (`echo "{\"cancel\":false,...}"`) to
single-quoted ones (`echo '{"cancel":false,...}'`), which is the real
"shell escapes" fix. Second — and much larger in scope — it rewrites
the hook input contracts: `tool` → `toolName`, `durationMs` →
`executionTimeMs`, `taskStart.task` →
`taskStart.taskMetadata.initialTask`, `userPromptSubmit` gains an
`attachments: string[]` field, and `preCompact` is expanded from
`{conversationLength, estimatedTokens}` to a 13-field shape with
`taskId`, `ulid`, `contextSize`, `compactionStrategy`, token I/O
counts, deleted-range markers, and context paths.

## Reviewable points

- The shell-escape fix is correct and the right shape: the prior
  `"{\"cancel\":false,...}"` string was technically valid but fragile
  if anyone added a `$` or backtick inside the JSON, and the
  single-quoted form is the standard shell idiom.
  See `docs/customization/hooks.mdx:294`, `:347`, and the matching
  template strings in the JS file (e.g. `getPostToolUseTemplate`,
  `getUserPromptSubmitTemplate`, `getNotificationTemplate`,
  `getPreCompactTemplate`).
- The contract changes are **breaking**: any user with a hook script
  reading `.preToolUse.tool`, `.postToolUse.durationMs`,
  `.taskStart.task`, or `.preCompact.conversationLength` will silently
  get `null` from `jq -r` after this change. The mdx note around line
  240 mentions one prior workspacePath rename but does not call out
  this round of renames as a migration. A `## Breaking Changes`
  block in the docs would be high-value.
- `preCompact` going from a 2-field to a 13-field input is a
  significant expansion. The new shape includes `contextJsonPath` and
  `contextRawPath`, which are presumably absolute filesystem paths the
  hook can read; this should be documented as a security/PII
  consideration (hooks can now read full conversation context off
  disk).
- The internal `previousApiReqIndex` field is exposed in the docs but
  isn't obviously useful to script authors — consider either dropping
  it from the documented contract or adding a one-line description.

## Rationale

The shell-escape change itself is a clear win. The contract migration
is also justifiable (the new field names match runtime types better),
but bundling two unrelated changes under a "fix" title masks the
breaking-change surface. Splitting into two PRs would be ideal; if
that's not feasible, at minimum add a migration table to the docs and
flip the PR title/description to reflect the breaking nature.
