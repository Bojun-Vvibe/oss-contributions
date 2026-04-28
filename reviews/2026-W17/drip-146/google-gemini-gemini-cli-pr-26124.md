# google-gemini/gemini-cli #26124 — fix(patch): cherry-pick 54b7586 to release/v0.40.0-preview.4-pr-26066 [CONFLICTS]

- PR: https://github.com/google-gemini/gemini-cli/pull/26124
- Head SHA: `a5c6b7d7cb8a6c5e7fd6873172333816d0fbacce`
- Author: gemini-cli-robot (automated cherry-pick bot)
- Files touched: `docs/reference/configuration.md`, `packages/core/src/availability/policyCatalog.test.ts`, `packages/core/src/availability/policyCatalog.ts`, `packages/core/src/config/defaultModelConfigs.ts`, `schemas/settings.schema.json`

## Observations

- This is an automated cherry-pick of upstream `54b7586106bb3431080978be8960ab9a8d3cf0d8` onto `release/v0.40.0-preview.4-pr-26066`. The PR body itself acknowledges merge conflicts and tells a human to resolve them — i.e. this PR is *not* in a mergeable state and is awaiting manual conflict resolution.
- `docs/reference/configuration.md:1154`, `:1170`, `:1187`, `:1203`, `:1220`, `:1235`, … — at every per-policy `stateTransitions` block in the reference docs, `"transient": "terminal"` flips to `"transient": "sticky_retry"`. This is the substantive behavior change being backported: transient errors no longer hard-terminate, they enter a sticky-retry state. Several dozen identical edits across the policy catalog reference.
- `packages/core/src/availability/policyCatalog.ts` (and its test) — not visible in the head of the diff but presumably mirror the transition table change so runtime matches the docs.
- `packages/core/src/config/defaultModelConfigs.ts` — likely the conflict surface (model defaults often diverge between release and main).
- The PR is procedurally correct: bot opened the cherry-pick, marked it `[CONFLICTS]`, did not auto-merge. Reviewer attention should focus on (a) does the resolved diff match the original `54b7586` semantically and (b) the `defaultModelConfigs.ts` resolution didn't drop any release-line model entries.

## Verdict: `needs-discussion`

**Rationale:** The change itself (transient → sticky_retry) is desirable for release stability, but this PR is a `[CONFLICTS]` automated cherry-pick that nobody has resolved yet — merging in its current state would land conflict markers. A maintainer needs to either resolve manually, force-push the resolution, or close in favor of a hand-crafted backport. Cannot merge until conflicts are addressed.
