# sst/opencode#25081 — fix(share): render all turns as scrollable list instead of single-turn + nav

- URL: https://github.com/sst/opencode/pull/25081
- Head SHA: `d2ba379b4fff603308bf2ab2ec6e94d96a3e3fd4`
- Size: +23 / -40 (1 file)

## Summary

Replaces the share page's single-turn-with-`MessageNav` UX in `packages/enterprise/src/routes/share/[shareID].tsx` with a scrollable `<For each={messages()}>` list rendering every `SessionTurn` inline. The local `solid-js/store` `[store, setStore]` (`messageId`) state and the `setActiveMessage`/`activeMessage` indirection are deleted; `provider`/`modelID` are now derived from `firstUserMessage()` directly. Net `-17` lines of state machinery for what is functionally a layout simplification.

## Observations

- `share/[shareID].tsx:9` removes the `createStore` import; `:14` removes `MessageNav`. Lines `:191-198` collapse the `activeMessage`/`setActiveMessage` pair down to two `createMemo`s on `firstUserMessage()`. The deletion is total — there is no fallback for the previous nav-driven `provider`/`modelID` selection (which previously could differ per message). For a public share view this is the right call; just confirm that long sessions where the user changed model mid-thread still display sanely (the header now always reflects the first turn's model regardless of which turn the reader has scrolled to).
- The new list shell at `:288-296` wraps the `<For>` in `flex-1 overflow-y-auto no-scrollbar px-6 min-h-0`. The `no-scrollbar` utility hides the scrollbar entirely on a tall scroll surface — verify this was intentional (silent scroll on a long share with no visible affordance is a known accessibility/UX pitfall vs. the old explicit nav).
- `SessionTurn` is now invoked with `messageID={message.id}` per turn (`:302`) and the per-turn `container: "w-full pb-20 px-6"` becomes `container: "w-full"` with the horizontal padding lifted to the outer scroll container — clean separation of layout concerns. The `<Logo>` is now a single trailing decoration (`:295`) rather than per-turn.
- No test changes in the diff. The previous `MessageNav` integration likely had no behavioral test either, so removing it doesn't visibly regress coverage, but a snapshot/visual test for the scrollable share view would lock the new layout.

## Verdict

**merge-after-nits**

## Nits

- Confirm the per-turn-model header drift is acceptable, or thread the active scroll position back into `provider`/`modelID` derivation via `IntersectionObserver`.
- `no-scrollbar` on a `flex-1 overflow-y-auto` surface should at minimum show on hover or thumb-on-scroll for discoverability.
- A one-line CHANGELOG entry would help downstream consumers who depended on the per-message deep-link behavior (the `store.messageId` was effectively a query-param target).
