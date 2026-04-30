# block/goose#8928 — Lifei/UI improvements plan

- URL: https://github.com/aaif-goose/goose/pull/8928
- Head SHA: `5249b5594bb9bda37699d2da6e75c2afa3a3f3de`
- Size: +2483 / -0 (docs-only)

## Summary

Adds a `+569`-line architecture-review markdown file at `ui/goose2/ui_improvements/goose2-acp-session-id-architecture-review.md` plus (judging from the +2483 total) several other docs/RFC files in the same directory. The visible file is a deep, well-structured analysis of the goose2 frontend's ACP session-id handling, arguing the canonical invariant `ChatSession.id == ACP sessionId == Goose Thread.id` and laying out a layered cleanup plan (UI store, ACP transport, runtime tracker) plus a list of named architectural smells (failure-driven control flow in `acpCreateSession()`, overloaded `prepareSession()`, redundant `acpSessionId` field that duplicates `ChatSession.id`, etc.).

## Observations

- `goose2-acp-session-id-architecture-review.md:1-90` (Domain Model section) lays out the right invariant and traces it through both backend (`Thread 1 : many internal Sessions`) and frontend (`messagesBySession[ChatSession.id]`, `draftsBySession[ChatSession.id]`, `activeSessionId: ChatSession.id`). The framing — "internal `Session.id` is backend-private runtime state and should not cross the goose2 UI boundary" — is the load-bearing thesis and is correctly grounded in the original Thread-introduction commit quoted at `:51-58`.
- `:130-155` (Architectural Risk section) names the actual smell precisely: `acpCreateSession()` synthesizes a `crypto.randomUUID()` local id, hands it to `acpPrepareSession(localSessionId, ...)`, which then **relies on `loadSession(localSessionId)` failing** before falling through to `newSession()`. Failure-driven control flow as a session-creation primitive is a real bug surface — any future change to `loadSession`'s error semantics (caching a NotFound response, returning a sentinel object, retrying) would silently break session creation. The doc correctly flags this.
- `:154-165` enumerates 6 specific issues. Items 4 and 5 (the `gooseSessionId` naming is misleading because the value is actually ACP `sessionId` / `Thread.id`, and `ChatSession.acpSessionId` duplicates `ChatSession.id`) are the kind of latent ambiguity that breeds bugs at refactor time. The recommendation at `:189-191` ("remove `ChatSession.acpSessionId` by end of cleanup; it's only a temporary compatibility field") is the right end-state.
- This is a docs-only PR (`ui/goose2/ui_improvements/`) and intentionally does not modify code. As an RFC / architectural-review artifact landing ahead of an implementation series, that's the right shape — but a `+2483` docs-only PR with no companion tracking issue and no enumeration of which other markdown files are included risks becoming a stale artifact six months from now if the cleanup series is deferred. The PR body should at minimum (a) link to the GH issue/project tracking the cleanup series, (b) name the expected first implementation PR, (c) name an owner who will close the doc out when the cleanup is complete or supersede it. Without those hooks this becomes architectural-debt-as-documentation.

## Verdict

**needs-discussion**

## Nits

- File path under `ui/goose2/ui_improvements/` rather than under a top-level `docs/architecture/` or `rfcs/` directory will make it hard to find later — recommend lifting the planning docs to `docs/rfcs/goose2-session-id-cleanup/` or similar.
- PR description should enumerate the other markdown files in the +2483 (only one is visible in the diff slice).
- Add a tracking issue link and an explicit "supersedes / superseded by" footer so the doc has a defined lifecycle.
- Confirm an owner and an expiration condition (e.g. "delete this file when `ChatSession.acpSessionId` is removed from the codebase").
