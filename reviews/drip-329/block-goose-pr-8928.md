# block/goose #8928 — Lifei/UI improvements plan

- SHA: `5249b5594bb9bda37699d2da6e75c2afa3a3f3de`
- State: OPEN, +2,483/-0 across 5 files
- Files: `ui/goose2/ui_improvements/goose2-acp-session-id-architecture-review.md` (+569), `ui/goose2/ui_improvements/path_manipulation/goose2-path-manipulation-implementation-plan.md` (+501), `ui/goose2/ui_improvements/path_manipulation/goose2-path-manipulation-review.md` (+594), `ui/goose2/ui_improvements/state_management/goose2-zustand-state-management-improvement-plan.md` (+435), `ui/goose2/ui_improvements/state_management/goose2-zustand-state-management-review.md` (+384)

## Summary

Pure docs PR. Drops 5 markdown design / review documents under `ui/goose2/ui_improvements/` covering: ACP session-id architecture review, path-manipulation implementation plan + review, and Zustand state-management improvement plan + review. No code changes, no tests, no config.

## Notes

- `ui/goose2/ui_improvements/goose2-acp-session-id-architecture-review.md:1-50` — argues that `ChatSession.id == ACP sessionId == Goose Thread.id` should be the single canonical identity in goose2 frontend, and that the current temporary local-id + local-to-ACP mapping layer is unnecessary risk. The argument is well-structured: starts with backend domain model (ACP session ↔ Goose Thread vs. Goose internal Session.id), then cites the original thread-introduction commit's rationale ("Threads are a new more durable abstraction... append-only history of the messages seen by the end user"), then walks the current implementation. Clear and actionable.
- `goose2-acp-session-id-architecture-review.md` — the document is a *review* artifact, not an implementation plan. It identifies the architectural debt (temporary local id surviving in the create/prepare path) but does not gate or schedule the cleanup. As a planning doc this is fine; as a tracking artifact it should reference an issue number for the cleanup work.
- `goose2-path-manipulation-implementation-plan.md` (+501) and `goose2-path-manipulation-review.md` (+594) — paired plan + review. The fact that the review (594 LOC) is longer than the plan (501 LOC) suggests post-implementation analysis, which means actual code changes likely already exist in goose2. This PR doesn't include those code changes — recommend cross-linking to the corresponding code PR(s) so future readers can navigate from doc → implementation.
- `goose2-zustand-state-management-improvement-plan.md` (+435) and `goose2-zustand-state-management-review.md` (+384) — same pattern. Plan + review for a Zustand store reorganization. Same cross-link recommendation.
- All 5 files land under `ui/goose2/ui_improvements/`, a new directory tree. Worth confirming with maintainers whether design docs for goose2 belong here or under a top-level `docs/` / `ui/goose2/docs/` location. Co-locating with the code is fine but if the project has a docs convention this should match it.
- No `README.md` index for the new directory. With 5 substantive docs and likely more to come, a one-page index pointing at each would help discoverability.
- No code change → no test change required. CI risk is zero.
- The PR title `Lifei/UI improvements plan` reads like a working-branch name rather than a final commit subject. Recommend renaming on merge to something like `docs(goose2): add UI improvement plans and reviews` so `git log` is searchable.
- The PR body is the GitHub template with all sections still showing placeholders (`Relates to #ISSUE_ID`, empty Before/After). For a 2.5k-LOC docs PR this is the primary source of context for future readers — should be filled in before merge.

## Verdict

`merge-after-nits` — content is high-quality and useful. Block on: (1) PR title cleanup, (2) PR description filled out with the tracking issues / related code PRs, (3) one-line `README.md` or directory index in `ui/goose2/ui_improvements/` so the docs are discoverable. No risk to runtime, so the bar is just hygiene.
