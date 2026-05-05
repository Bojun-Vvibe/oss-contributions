# google-gemini/gemini-cli PR #26464 — fix(config): ensure configuration persistence and fix in-memory regressions

- Repo: `google-gemini/gemini-cli`
- PR: https://github.com/google-gemini/gemini-cli/pull/26464
- Head SHA: `cc2076c36cc4`
- Size: +468 / -154 across 10 files (cli config + tests, core config + tests, settings dialog test, gemini.tsx wiring)

## Summary

Cross-cutting fix for four distinct settings-persistence bugs that
were combining to silently corrupt or revert user config. The PR (1)
unifies parsing on `comment-json` so the loader and saver agree on
trailing-comma legality, (2) tightens the migration code path so it
no longer stomps entire top-level sections during a partial in-memory
update, (3) plumbs an `onReload` callback from `loadCliConfig` into
the core `Config` so settings that change mid-session — model name,
compression threshold, IDE mode, context-management toggles — pick
up the new values without restart, and (4) backs the lot with new
tests in `cli/config/config.test.ts`,
`cli/config/settings.test.ts`, `cli/ui/components/SettingsDialog.test.tsx`,
and `core/src/config/config.test.ts`.

## What I like

- The parser-mismatch story in the PR description is concrete and
  matches what I'd expect to see in support traffic: loader strict,
  saver lenient → trailing-comma settings file silently loads as `{}`
  → next save persists `{}` → user's config "vanished." The fix
  (`utils/commentJson.ts:+3/-2` exports `parse`; `settings.ts` and
  the test harness now both go through it) is the obvious right
  answer. The new mock in `settings.test.ts:170-180`
  (`commentJsonParseMock`) lets tests assert against the unified
  parser instead of the previous JSON-spy hack.

- The migration tightening is in `settings.test.ts:2240-2284`. The
  diff shows the assertions changing from
  `setValueSpy.toHaveBeenCalledWith(SettingScope.User, 'general',
  expect.objectContaining({ enableAutoUpdate: false }))` to
  `setValueSpy.toHaveBeenCalledWith(SettingScope.User,
  'general.enableAutoUpdate', false)`. So the migration path now
  writes the leaf key path directly instead of replacing the entire
  `general` object — that's the actual fix for sibling keys getting
  deleted when a partial in-memory `general` was passed in. A
  `deleteValueSpy` is also introduced (line 2264) which suggests the
  migration now properly distinguishes "set this leaf" from
  "delete this deprecated leaf."

- The `onReload` callback plumbing at `cli/src/config/config.ts:1126-1144`
  is the cleanest part of the PR. The old return was just
  `{ disabledSkills, agents }`; the new return adds a `settings` block
  exposing `model`, `compressionThreshold`, `ideMode`,
  `contextManagement.enabled`, `topicUpdateNarration`,
  `experimentalAutoMemory`, `experimentalMemoryV2` plus
  `adminSkillsEnabled`. These are exactly the surfaces a user
  flipping a toggle in `SettingsDialog` would expect to take effect
  immediately without restart. The shape (a flat-ish settings dict
  vs the deeply-nested `merged.experimental.contextManagement`) is
  also right — flatten at the API boundary so the core doesn't need
  to know the cli's nesting scheme.

- The test in `config.test.ts:4040-4082` exercises the round trip:
  initial settings → call `loadCliConfig` → grab `onReload` from
  `ServerConfig.Config` mock → mock `loadSettings` to return
  refreshed values → call `onReload` → assert flat fields on the
  result. The assertions cover all six new settings paths
  (`compressionThreshold`, `ideMode`, `contextManagement.enabled`,
  `topicUpdateNarration`, `experimentalAutoMemory`,
  `experimentalMemoryV2`). That's the right shape for "I added a
  field to the contract; pin every field."

- The error-path test for invalid JSON
  (`settings.test.ts:1233-1281`) was rewritten to use
  `commentJsonParseMock.mockImplementationOnce` instead of the old
  `vi.spyOn(JSON, 'parse')` hack. Cleaner, and matches the unified-
  parser invariant the production code now relies on.

## Nits / discussion

1. **`onReload` is fire-and-forget — what calls it?** The diff shows
   `onReload` being constructed and passed to `ServerConfig.Config`,
   but I don't see (in this slice) the wiring on the consumer side
   that actually invokes `onReload` when the user saves settings. If
   `Config` exposes `triggerReload()` or watches the file via fs
   events, fine — but the PR description should be explicit about
   what triggers a reload. Otherwise this looks like "the settings
   dialog still has to call X explicitly" which is a regression
   waiting to happen.

2. **The new `settings` payload duplicates what's already in
   `merged`.** `cli/config/config.ts:1129-1142` builds a flat
   `{ model, compressionThreshold, ideMode, ... }` from `merged.*`.
   Anyone who later adds a new setting now has to remember to add
   it both to the schema and to this manual flattening (or it
   silently won't reload). A schema-driven flatten — or just passing
   `merged` through and letting core consumers project — would scale
   better. As written, this is a manual checklist that will rot.

3. **`adminSkillsEnabled: merged.skills.enabled`** was added to the
   payload (line 1132). The `disabledSkills` field already shipped.
   So now the API exposes both "enabled" and "disabled" — which one
   is canonical when both are populated? Worth a doc comment, or
   pick one and derive the other.

4. **Test mock surface keeps growing.** `Config:
   vi.fn().mockImplementation((params) => new actualServer.Config(params))`
   at `config.test.ts:169` is added so the test can intercept
   constructor params. Reasonable, but it means the mock is now
   "real Config + spy" which is a fragile arrangement — the spy
   gives you the constructor args but any internal state mutation
   in the real Config still happens. A dedicated injection point
   (e.g. `loadCliConfig({ onReloadSink: ... })`) would be cleaner
   for tests than mocking the constructor. Out of scope for this
   PR.

5. **PR scope is large and crosses cli/core.** The four bugs are
   genuinely related (they all manifest as "my settings aren't
   sticking"), so single-PR makes the user-visible story
   reviewable. But four separate fixable bugs in one diff means
   bisecting any future regression is harder. If splittable
   without rebase pain, splitting parser-fix / migration-fix /
   onReload-extension into three commits would help future
   bisects.

6. **No changelog entry visible in the diff.** A user-visible
   "settings now reload mid-session" + "trailing commas in
   settings.json no longer wipe your config" line in the
   changelog would help users understand why their behavior
   changed.

## Verdict

**merge-after-nits.** Real fixes for a real persistence-corruption
class of bugs (parser mismatch is the showstopper; aggressive
migration is the silent data-loss one). The `onReload` extension is
the right architectural shape. Tests are pointed at the failure
modes rather than at coverage % alone. Before merge: (a) clarify in
the PR description what triggers `onReload` invocation
(file-watch? explicit save callback?) so reviewers can verify the
end-to-end path, (b) add a comment or schema-driven flatten so
future settings don't silently miss the reload contract, and
(c) write a changelog entry. None of these block.
