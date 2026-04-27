# google-gemini/gemini-cli PR #25966 — add security warning to eval-gate approval comment

- **PR**: https://github.com/google-gemini/gemini-cli/pull/25966
- **Author**: @herdiyana256
- **Head SHA**: `8dd83b5803713a6daa242db24a87e9cf1ad090e8`
- **Size**: +9 / −6
- **Files**: `.github/workflows/eval-pr.yml`

## Summary

Edits the GitHub Actions comment posted by `eval-pr.yml` when steering changes are detected in a fork PR. The new comment hardens the language: it relabels the title from "Action Required" to "[SECURITY WARNING]: Evaluation Approval Required", adds a "[CRITICAL]: Approving this run will execute code from the fork on Google infrastructure with access to the `GEMINI_API_KEY` secret" line, and ends with "**DO NOT APPROVE** unless you have fully reviewed all changes... A malicious PR can steal the API key or compromise the CI environment."

## Verdict: `merge-as-is`

The original prompt under-described the threat model. A maintainer who sees "Action Required: Evaluation Approval" and clicks "Approve deployments" without inspecting the fork's `package.json` / `scripts:` / source changes is one PR away from leaking `GEMINI_API_KEY`. Spelling that out at the point of decision is the textbook mitigation for "approval fatigue" CI patterns and costs nothing.

## Specific references

- `.github/workflows/eval-pr.yml:62-77` — the entire change is in the heredoc-style multi-line `COMMENT_BODY` shell variable. The structural changes are: (1) the `### 🛑 Action Required:` header is replaced with `### [SECURITY WARNING]: Evaluation Approval Required`, (2) two new paragraphs name the secret (`\`GEMINI_API_KEY\``) and the failure mode ("execute code from the fork on Google infrastructure"), and (3) a `**DO NOT APPROVE** unless...` admonition is added before the existing 3-step approval instructions.
- `:77` — the trailing `<!-- eval-approval-notification -->` HTML comment marker is preserved, so the existing `if comment_already_exists` dedup at the bottom of the run-script (referenced but not shown in the head of the diff) keeps working. Critical — without that marker, this PR would post a duplicate comment on every workflow re-run.
- `:62-67` — note that the indentation went from leading-tab to leading-4-space in three of the new lines, but bash heredoc-quoted strings are whitespace-preserving, so the rendered Markdown will show literal indentation. On GitHub, that renders as a code block, not as paragraph text. **This is the one issue worth flagging**: the ` ` (4-space) prefix on lines 64, 66, 68, 70, 72, 74, 76 will turn each of those lines into an indented Markdown code block. The original used tab characters for those lines. Either use the same tab indentation as the original, or strip leading whitespace consistently. This will look broken in the rendered comment.

## Nits

1. **Whitespace regression** (see ref above): the new lines use 4-space indentation where the surrounding bash kept tabs. GitHub Markdown will render the new paragraphs as indented code blocks, defeating the formatting. Re-check the rendered output on a draft PR before merging — if it renders as a single code-block lump, strip the leading 4 spaces on the new prose lines.
2. The `[SECURITY WARNING]` and `[CRITICAL]` labels-in-brackets are non-standard; the existing convention in the repo uses emoji + bold (`### 🛑 ...`). If the security-team's intent is to make these searchable in audit logs, the bracket form is fine and even desirable. If it's purely visual, the emoji form is more consistent.
3. Consider linking the actual GitHub Actions secret-scope docs (`https://docs.github.com/en/actions/security-guides/encrypted-secrets`) so the maintainer who's never thought about fork PRs running with secrets has a primary source one click away.

## What I learned

The "fork PR runs with maintainer-controlled secrets" footgun is one of the most-exploited CI patterns in OSS, and the mitigation is mostly social: the approving maintainer must understand exactly what they're authorizing. A boilerplate "approve deployments" prompt loses that signal; a comment like this one regains it. The cost is a 9-line YAML diff. The price of getting it wrong is a full credential rotation across whatever scope `GEMINI_API_KEY` opens. Always worth merging this class of fix, with one round of "is the indentation actually rendering right" sanity checking.
