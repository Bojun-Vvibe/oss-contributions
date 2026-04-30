# Review: aaif-goose/goose#8928 — Lifei/UI improvements plan

- PR: https://github.com/aaif-goose/goose/pull/8928 (originally listed under block/goose#8928 — repo
  has been transferred to aaif-goose org)
- Author: lifeizhou-ap (Lifei Zhou)
- headRefOid: `5249b5594bb9bda37699d2da6e75c2afa3a3f3de`
- Files: 5 new markdown design docs in `ui/goose2/ui_improvements/` (+2483/-0)
  - `goose2-acp-session-id-architecture-review.md` (569)
  - `path_manipulation/goose2-path-manipulation-implementation-plan.md` (501)
  - `path_manipulation/goose2-path-manipulation-review.md` (594)
  - `state_management/goose2-zustand-state-management-improvement-plan.md` (435)
  - `state_management/goose2-zustand-state-management-review.md` (384)
- Verdict: **needs-discussion**

## Analysis

Pure-docs PR adding ~2.5k lines of architecture review and improvement plans for the goose2 UI
covering three themes: (1) ACP session-id canonicalization, (2) path-manipulation abstraction
in the renderer, (3) zustand state-management refactoring. The content I sampled in
`goose2-acp-session-id-architecture-review.md` is high quality — it correctly identifies that
`ChatSession.id == ACP sessionId == Goose Thread.id` should be the only frontend session identity,
walks through `acpCreateSession()` / `acpPrepareSession()` / `acpSessionTracker.prepareSession()`,
and points at the architectural risk where a temporary `crypto.randomUUID()` local id is created
even though it's currently rescued by the `loadSession` → `newSession` fallback. The persistence
analysis around `goose:home-session-id` and `goose:chat-drafts` is also accurate.

**Why "needs-discussion" instead of merge-as-is:**

1. **Scope and review surface.** Five large planning documents shipped in a single PR is hard to
   review independently — each of the three themes would be more useful as its own PR so that
   reviewers from different areas (ACP transport, renderer, state) can engage without context-
   switching across 2.5k lines. The PR body is also empty (just template placeholders), so future
   readers landing in `git log` will have to open all five docs to figure out what was decided.

2. **Status of the docs.** It's not clear from the diff whether these are *approved* designs the
   team has converged on, or *proposals* that still need internal sign-off. The session-id
   architecture review reads like an approved canonical doc ("Goose2 should use one canonical
   frontend session id"), but the improvement plans read more like RFCs. Mixing accepted
   architecture and open proposals in the same `ui/goose2/ui_improvements/` tree will confuse
   future contributors about which is normative.

3. **Implementation follow-up.** The session-id review explicitly identifies `acpCreateSession()`
   creating a throwaway `localSessionId` as architectural risk. Landing the doc without a tracking
   issue or follow-up PR means the risk persists with no owner. Either link existing issues or
   open follow-ups before merge so the docs aren't write-only artifacts.

The content itself is good — I'd want this knowledge in the repo. Splitting into three smaller
PRs (one per theme), filling in the PR body with a one-paragraph context per doc, and either
linking implementation tracking issues or labeling docs `proposal/` vs `accepted/` would unblock
me. Until then: needs-discussion.
