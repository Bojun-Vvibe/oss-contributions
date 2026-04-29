---
pr: google-gemini/gemini-cli#26207
sha: 16ff1e342a67de573d2301ebe22d27e9ed96daf5
verdict: request-changes
reviewed_at: 2026-04-29T18:31:00Z
---

# Add the ability to @ mention the gemini robot

## Context

Extends `.github/workflows/gemini-cli-bot-brain.yml` to fire on
`issue_comment` events when a maintainer `@gemini-cli-robot` mentions
the bot. The bot then loads `tools/gemini-cli-bot/brain/interactive.md`,
runs in `ENABLE_PRS=true` mode, and is permitted to post comments and
push to `bot/`-prefixed branches via the
`GEMINI_CLI_ROBOT_GITHUB_PAT` secret.

## What's good

- The trigger guard at the job level is the right shape — uses
  `contains(fromJSON('["COLLABORATOR", "MEMBER", "OWNER"]'), github.event.comment.author_association)`
  to gate on association rather than username, which is harder to
  spoof. The reasoning job is also `permissions: contents: read`,
  so the LLM call itself can't push.
- `concurrency.group` now includes `github.event.issue.number || …`
  so two concurrent maintainer mentions on different issues don't
  cancel each other.
- The `<untrusted_context>...</untrusted_context>` wrapper around
  the comment body and `gh issue view` output in `trigger_context.md`
  is the correct prompt-injection mitigation idiom, and concatenating
  it *before* the system prompt keeps untrusted text from clobbering
  later instructions.
- The push-time branch guard (`if [[ ! "$BRANCH_NAME" =~ ^bot/ ]]; then exit 1; fi`)
  and the comment-time author guard (`PR_AUTHOR=$(gh pr view "$PR_NUM" --json author --jq '.author.login'); if [ "$PR_AUTHOR" != "gemini-cli-robot" ]; then exit 1; fi`)
  are the right safety belt for the publish phase.

## Concerns

1. **Critique semantics inverted.** The previous code rejected on
   `[REJECTED]` *or* non-zero exit, defaulted to approve. The new
   code rejects on anything that isn't *explicitly* `[APPROVED]` and
   also lacks `[REJECTED]`. That's a stronger gate (good), but the
   comment "Critique failed, rejected, or did not explicitly approve
   changes" hides the fact that this is now a fail-closed contract.
   Maintainers who relied on the old "neutral = approve" semantics
   for their own dispatched runs will see silent skips. Worth a
   release note.
2. **`Post PR/Issue Comment` step lost its `if:` guard.** Old code
   had `if: "${{ github.event.inputs.enable_prs == 'true' }}"`. The
   new step runs unconditionally — it's safe because the inner `if [ -s ... ]`
   tests check for content, but on a scheduled run with no comment
   artifact you now do extra work and the step shows green-with-noop
   in the UI. Minor but a regression in clarity.
3. **`gh issue view ... 2>/dev/null || gh pr view "$TRIGGER_ISSUE_NUMBER"`**
   — if both fail (deleted issue, permissions hiccup), the script
   keeps going with empty trigger context. The bot then operates
   blind on whatever `interactive.md` says by default. Should
   `set -e` or explicitly check `[ -s trigger_context.md ]` before
   proceeding.
4. **PAT scope.** `GEMINI_CLI_ROBOT_GITHUB_PAT` now ships into a
   workflow that any maintainer comment can trigger. If a
   compromised maintainer account exists, this is a remote command
   execution into the bot's branch space. The `bot/` prefix limits
   the blast radius but doesn't eliminate it. Consider narrowing
   the PAT to `contents: write` on `refs/heads/bot/*` only via a
   GitHub App installation token rather than a long-lived PAT.

## Verdict

`request-changes` — the security model is mostly sound, but the
critique-result semantics flip needs an explicit comment in the
diff, the comment-step `if:` guard should be restored, and the
unconditional fall-through when both `gh issue view` and `gh pr view`
fail needs to abort. The PAT-vs-App-token discussion is a separate
follow-up but worth raising before this lands.
