# sst/opencode#24894 — docs(security): add static security review report for dev branch

- URL: https://github.com/sst/opencode/pull/24894
- Head SHA: `d8d039cdfb8615d148ceda70224028a6320c0861`
- Size: +260 / −0, 1 file (`.github/security-reports/static-analysis-5aba522a-243c-465a-ad62-a48d7a756ae3.md`)
- Verdict: **needs-discussion**

## Summary

Adds a static-analysis security report (101 findings, risk score 100/100) as a markdown doc under `.github/security-reports/`. No code changes, no automatic fixes — every finding is flagged for human review. Highlighted classes: F-001/F-002 alleged Anthropic API key in `github/README.md:120`; F-003 to F-005 dynamic tool selection from user input in `acp/agent.ts`, `session/llm.ts`, `session/processor.ts`; F-006 to F-012 unguarded "use server" Server Actions across billing/auth routes; high-severity `pull_request_target` workflow uses in `.github/workflows/{pr-management,pr-standards,vouch-check-pr}.yml:4`.

## Specific issues flagged

### 1. F-001/F-002 needs verified-or-revoked status before this lands

Report at lines 22-37 claims a real Anthropic-format secret at `github/README.md:120` with 94% confidence. **This PR cannot land while that finding is unresolved** — either:

- The secret is real, in which case it's already been pushed to the public history of `dev` (and possibly published) and needs `git filter-repo` + AWS/Anthropic-side rotation before *any* further commits are made to `dev`, including this docs PR;
- It's a false positive (e.g., a code fence with a placeholder), in which case the PR should annotate F-001/F-002 with "VERIFIED FALSE POSITIVE — placeholder, see context at github/README.md:115-125" so this report doesn't accidentally trigger downstream secret-scanning workflows that block the next 100 PRs.

Cannot approve a security report that asserts an exposed credential without the resolution receipt. Even as docs, this becomes a "broken window" — if F-001 is ignored once, every subsequent finding loses weight.

### 2. F-003 to F-005 are flagged but the report is filed against `dev` HEAD which is a moving target

Report header at lines 7-12: `Source: github.com/anomalyco/opencode#dev`, `Scan ID: 5aba522a-...`. The `acp/agent.ts`, `session/llm.ts`, `session/processor.ts` line numbers are pinned to *some* specific commit, but the report doesn't include the SHA of `dev` at scan time. Six months from now this report's line numbers are useless because the file moved. Add `Source SHA: <sha>` to the metadata block at lines 5-12, and ideally add a one-line note at the top of every finding ("at SHA `<sha>`") so future readers can `git show <sha>:acp/agent.ts | sed -n '28p'` to see what was actually flagged.

### 3. The `.github/security-reports/` location is the wrong place for human-review-required findings

Two problems with putting this here:

- `.github/` directory in many GH Actions setups is auto-included in `pull_request_target` workflow contexts (which the report itself flags as risky in F-007 et al). Adding 260 lines of plaintext "here is where the secrets are" to a `pull_request_target`-readable path creates a meta-vulnerability where a malicious fork PR could grep this report to harvest the curated list of weak spots before exploit.
- 101 findings as one markdown doc is unactionable. There's no issue tracker integration, no per-finding "owner", no "FIXED in #XXXX" closure mechanism. Within a week this report is stale and ignored.

Suggest: convert each finding into a GitHub issue (or a single tracking issue with a checkbox per finding), keep the report file in `docs/security/internal/` (path that is *not* exposed to fork-PR contexts), and link from the issue back to the file.

### 4. F-007 to F-012 (Server Actions missing auth guard) — confidence 84% is "likely false positive at this rate"

Auto-static-analysis "missing auth guard" on Next.js Server Actions almost always over-fires because guards are commonly hoisted to a parent layout, middleware, or RSC boundary that the analyzer doesn't see. Six findings at 84% confidence with no manual triage is exactly the noise-to-signal ratio that gets reports ignored. Either:

- Annotate which of F-007/F-008/F-009/F-010/F-011/F-012 the maintainer has actually verified vs. which are still unreviewed;
- Or remove the unverified ones from this initial drop and re-add them in a follow-up after triage.

A 101-finding report where 0 are triaged is worse than a 10-finding report where all 10 are triaged.

### 5. Risk score "100/100" with no methodology link

Lines 11-19: `Risk score: 100/100`, `Runtime / agent risk: Critical (100/100)`, `Secrets risk: Critical (83/100)`. There's no link to how 100/100 is computed or what the rubric is. If the scoring is "any critical finding maxes the score", that's worth saying inline so 100/100 doesn't read as "every aspect of this codebase is maximally insecure."

## Why needs-discussion

Items 1 and 3 are blockers that need maintainer + security-team sign-off, not nits an author can address in a follow-up commit. F-001 in particular cannot be left ambiguous — the answer is either "rotated and history-cleaned" or "false positive, here's why" *before* this PR proceeds.
