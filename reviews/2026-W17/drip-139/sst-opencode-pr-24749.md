# sst/opencode #24749 — fix(ui): remove redundant flex overrides in tool components

- PR: https://github.com/sst/opencode/pull/24749
- Author: Brendonovich (Brendan Allan)
- Head SHA: `c89d562cc582`
- Base: `dev` · State: MERGED
- Diff: 0+/12- across 2 files (`packages/ui/src/components/basic-tool.css`, `packages/ui/src/components/message-part.css`)

## Verdict: merge-as-is

## Rationale

- **Pure deletion of dead-or-conflicting CSS.** The `[data-hide-details="true"]` block at `basic-tool.css:14-23` set `flex: 1 1 auto` + `max-width: 100%` on `basic-tool-tool-trigger-content`/`basic-tool-tool-info`. The block immediately after (`:25-`) sets `flex: 0 1 auto` + `width: auto` on the same `basic-tool-tool-trigger-content`. Two specificity-equal rules on the same selector with conflicting `flex-grow` values means the CSS-cascade winner depends purely on source order — and the `[data-hide-details="true"]` arm only fires when the parent has that attribute, so the *behavior* of the trigger content silently flipped between "shrink-to-content" and "grow-to-fill" based on a sibling-toggled attribute. Removing the override returns the layout to the single deterministic rule.
- **The `apply-patch-filename` deletion at `message-part.css:1209` is the right call.** That selector already has `min-width: 0; overflow: hidden; text-overflow: ellipsis` — the canonical "I am a truncatable label" recipe. Adding `flex: 1 1 auto` on top of that means the filename competes with siblings for grow-space rather than yielding to them, which is the opposite of what an ellipsizable label wants. Three remaining properties form a coherent contract; the fourth was fighting them.
- **Zero-net-line "delete-only" diff is hard to land safely without screenshots, but the structural argument carries it.** The `data-hide-details` arm and the unconditional arm were both targeting `data-slot="basic-tool-tool-trigger-content"` with the same specificity. Anyone reading the file in source order would see two rules for the same slot and have to mentally compute "which wins when the data attribute is set" — a footgun for the next person touching this CSS. Collapsing to one rule is a maintainability win even setting aside the conflict.

## Nits / follow-ups

- A `data-hide-details="true"` selector still exists elsewhere in the codebase if any consumer was *depending* on the flex-grow behavior in hide-details mode (e.g., for a long tool description to fill the row). Worth grepping `data-hide-details` across `packages/ui/src` to confirm no other rule replaces the deleted behavior — and if so, the visual regression would only show up on long tool descriptions in collapsed/hide-details mode, which a no-screenshot PR wouldn't catch.
- Zero test coverage for CSS layout regressions — but that's a structural OSS-CSS reality, not specific to this PR. A Playwright snapshot at `packages/ui/test/visual/tool-row.spec.ts` (if such a thing exists) covering "long tool name in hide-details mode" would be the catch-net for the next iteration of this kind of cleanup.
- Commit message could note *why* the override was redundant (i.e., conflicting with the unconditional rule below it) rather than just "redundant", to spare the next person the same investigation.

## What I learned

CSS "redundant override" deletions are a small-surface category that's easy to underestimate — the diff is tiny, the test surface is empty, and the tooling can't prove correctness. The structural tell is "two rules for the same selector at the same specificity with conflicting `flex` values": that's not redundancy, that's a *cascade-order-dependent toggle*, which is one of the worst maintainability shapes CSS can take. Deleting the override is correct precisely because it removes the cascade-order ambiguity, not because it removes "duplicate" code. The pattern generalizes: any time you see a parent-attribute-gated arm setting `flex-grow: 1` while the unconditional arm sets `flex-grow: 0`, treat it as a behavior switch hiding in styling code, not as styling duplication.
