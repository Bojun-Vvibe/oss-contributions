# BerriAI/litellm PR #27037 — merge main

- **PR:** https://github.com/BerriAI/litellm/pull/27037
- **Author:** Sameerlite
- **Head SHA:** `cfa058c3` (full: `cfa058c3e9425e5431bb84d4f33551efc5ff6cfa`)
- **State:** MERGED
- **Files touched (selected — diff exceeded 300 file API cap):**
  - `.github/workflows/create-release.yml` (+4 / -2)
  - `Makefile` (+3 / -0)
  - `docs/my-website/docs/providers/crusoe.md` (+196 / -0, new provider doc)
  - `enterprise/litellm_enterprise/proxy/auth/custom_sso_handler.py` (+32 / -22)
  - `enterprise/litellm_enterprise/proxy/management_endpoints/project_endpoints.py` (+54 / -29)
  - `litellm/_logging.py` (+3 / -66)
  - `litellm-proxy-extras/pyproject.toml` (+2 / -2)

## Verdict

**needs-discussion**

## Specific refs

- PR title literally `merge main` and head SHA `cfa058c3e9425e5431bb84d4f33551efc5ff6cfa` — GitHub's diff API rejects this PR with `HTTP 406: Sorry, the diff exceeded the maximum number of files (300)`. That alone is the review signal: a single merged PR touching >300 files with a body that's just the unfilled template ("Pre-Submission checklist", no relevant issues, no Linear ticket) is impossible to review on the PR page.
- `enterprise/litellm_enterprise/proxy/auth/custom_sso_handler.py` (+32 / -22) and `enterprise/litellm_enterprise/proxy/management_endpoints/project_endpoints.py` (+54 / -29) — auth and project-management endpoints in the enterprise tree changed inside a "merge main" PR. These are the kind of files where every diff line wants a per-PR review, not to be folded into a sync commit.

## Rationale

Mechanically this is a branch-sync PR — the title, the empty body, and the >300-file diff all point to "I rebased my branch onto main and re-opened the PR." That pattern is fine for personal forks but it's hostile to reviewers: the PR-level diff loses any signal about what the contributor actually authored vs what they pulled in from main, and CODEOWNERS-on-paths can't distinguish the two either. The substantive concern is that the touched paths include enterprise auth/SSO and project-management endpoints — anything in those files needs an explicit "this came from main, not from me" pointer with the upstream commit SHAs, otherwise an approving reviewer is implicitly LGTM-ing whatever happened to land in those files. Either re-open as a clean topic branch off the latest main, or file an inline maintainer note explaining which commits belong to the author and which are sync. Marking needs-discussion rather than request-changes because the PR is already merged; this is process feedback for the next round.

