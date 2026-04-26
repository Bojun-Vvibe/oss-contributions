---
pr: 24471
repo: sst/opencode
sha: 66837827db45
verdict: needs-discussion
date: 2026-04-26
---

# sst/opencode #24471 — feat: Add queued message editing, cancellation, and wrap-up behavior

- **URL**: https://github.com/sst/opencode/pull/24471
- **Author**: mortenfc
- **Head SHA**: 66837827db45
- **Size**: +596/-30 across 21 files (`packages/app/src/components/prompt-input/*`, `packages/app/src/pages/session/composer/*`, `i18n/en.ts`, `context/settings.tsx`, `ARCHITECTURE.md`, `STRUCTURE.md`)

## Scope

Three semi-independent UX changes packaged as one PR:

1. **Queued-message editing**: a queued follow-up can now be re-opened in the composer and re-submitted; the existing draft is replaced rather than appended (`submit.ts:282-318` adds a `queues followup and clears edit id when in normal mode and shouldQueue is true` test).
2. **Queue cancellation**: the followup dock surface picks up an explicit cancel/edit affordance (`session-followup-dock.tsx`).
3. **New `"wrap"` follow-up mode** alongside the existing `"queue"` and `"steer"`: settings union widens to `"queue" | "steer" | "wrap"` at `prompt-input.tsx:365` and the i18n string `settings.general.row.followup.option.wrap = "Wrap-up"` lands at `i18n/en.ts:405`.

Two new top-level docs (`ARCHITECTURE.md`, `STRUCTURE.md`) describing the Effect/monorepo layout are also bundled — their author attribution doesn't make it clear whether they're hand-written or LLM-generated.

## Specific findings

- **Three-feature PR is the headline issue.** "queued editing", "cancellation", and "wrap-up mode" are independently reviewable changes and each one wants its own discussion. The changelog will be hard to write, the bisect surface is wider than necessary, and a maintainer who wants only the cancellation fix has to take the wrap-up state machine too. Splitting into three PRs is the cleanest path; if not, the description needs a "what's in this PR and what's not" matrix.

- **`queueFollowup(draft, editID?)` signature change is the right shape** (`prompt-input.tsx:433`), but every caller needs to be checked: an existing callsite that passes a draft without `editID` now silently goes through the "no edit" branch, which is fine for new submissions but worth grepping the whole repo for `queueFollowup(` to make sure nothing was relying on the single-arg form for an implicit edit semantic.

- **`wrap` mode semantics are under-specified.** The diff at `prompt-input.tsx:419` adds `if ((mode === "steer" || mode === "wrap") && queuedFollowups().length >= 1) return false` — meaning wrap mode also enforces "at most one queued followup". Is that intentional? The PR description should state: "wrap-up mode behaves like steer for queue depth, but differs in X". From the i18n string alone ("Wrap-up") it's unclear whether the model is supposed to gracefully end the current turn after the queued message lands, or run the queued message immediately as the next turn, or something else.

- **Settings migration story is missing.** Existing users with `general.followup === "queue"` get mapped to `"steer"` in the legacy `setFollowup` (`prompt-input.tsx:386-391`) — that mapping is preserved, but a user whose persisted setting is `"queue"` literal will hit the widened union without an explicit migration. Confirm the persisted-settings codepath either tolerates `"queue"` going forward or auto-migrates on read.

- **`ARCHITECTURE.md` + `STRUCTURE.md` smell LLM-generated.** Sentences like "Pervasive use of `Effect.ts` for side-effect management, dependency injection (Layers/Context), and error handling" and the bullet-with-em-dash formatting throughout match common LLM doc-generation output. That's not disqualifying, but: (a) these need a maintainer's manual review for accuracy of paths and abstractions; (b) they should land in a separate "docs:" PR so that a future feature PR doesn't drag a stale architecture doc with it; (c) the `[project-root]/` placeholder in `STRUCTURE.md` is an obvious tell.

- **Test coverage**: only one new test (`submit.test.ts:282-318`) exercises the queue-edit path. No tests for: cancel-then-resubmit, wrap-mode-when-busy, wrap-mode-when-queue-full, or the i18n key being present in non-English locales (which will fail any locale-completeness CI).

- **i18n leakage**: only `en.ts` is touched. If sst/opencode has a translation completeness gate, this will fail; if it doesn't, every other locale silently falls through to the English key, which is acceptable but should be acknowledged.

- **`session-followup-dock.tsx` and `session-composer-region.tsx` changes** aren't visible in the head of the diff — reviewer should confirm the cancel UX is accessible (keyboard navigable, has aria-label on the cancel button, distinguishable from the existing dismiss action).

## Risk

Medium. Three concurrent changes to the central composer state machine, no rollback plan in the description, and the `"wrap"` mode is novel UX that users will form habits around — getting it wrong and changing it later costs trust. The doc files dragging along amplifies the diff-review burden.

## Nits

1. Split into three PRs (queued editing / cancellation / wrap-up mode) or write the matrix in the description.
2. Move `ARCHITECTURE.md` + `STRUCTURE.md` to a separate docs PR with maintainer-authored content.
3. Spec out `wrap` mode semantics in the description and as a JSDoc comment on the union type.
4. Add migration handling for the persisted `"queue"` literal value.
5. Add tests for cancel-then-resubmit and wrap-when-queue-full.
6. Grep all `queueFollowup(` callsites and confirm no implicit-edit reliance.

## Verdict

**needs-discussion** — three features in one PR plus two LLM-shaped doc files makes this hard to land cleanly, and the `wrap` mode semantics aren't pinned down enough to commit to. Maintainer should request a split. The underlying queued-edit work itself looks reasonable and would be a clean merge-after-nits on its own.

## What I learned

When a PR touches a state machine with three states (`"queue" | "steer" | "wrap"`), a single new state addition is the kind of change that *looks* tiny in the diff but actually doubles the testing matrix — every place that branches on the old enum needs to be re-examined for the third arm. Greppability matters here: the union type is the only authoritative source for "where do I need to add a `case`", and TypeScript's exhaustiveness checking only catches it when callsites use `switch`/`never`. PRs that widen a discriminated union should be reviewed with a "show me every callsite that branches on this" audit, not just the one new state's happy path.
