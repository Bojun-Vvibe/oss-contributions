# block/goose#9046 — feat(goose2): show built-in skills

- **Head SHA**: `c80fc8caa726c6f3616d585d18fd8ac45da1993d`
- **Stats**: +453 / -113, ~12 files (UI feature slice)

## Summary

Surfaces the new `builtinSkill` ACP source kind (added in #9045) inside the goose2 desktop UI. Adds a third `SkillSourceKind = "builtin"` alongside the existing `"global"` / `"project"`, fans the listing query out to `GooseSourcesList({ type: "builtinSkill" })` in parallel with the existing per-project queries, renders built-ins in a separate filter chip and group section, and gates the "edit / delete / link to project" actions off for builtin skills (they're read-only — they live inside the binary, not on disk).

## Specific citations

- `ui/goose2/src/features/skills/api/skills.ts:9` adds the `BUILTIN_SKILL_SOURCE_TYPE = "builtinSkill"` constant; `:18` extends `SkillSourceKind` to `"global" | "project" | "builtin"`. The discriminated-union split at lines 37-46 (`FilesystemSkillSourceEntry` vs `BuiltinSkillSourceEntry`) lets `toSkillInfo` switch on `source.type` rather than on a side-channel boolean — clean.
- `ui/goose2/src/features/skills/api/skills.ts:55-71`: the builtin branch of `toSkillInfo` constructs `id: \`builtin:${source.name}\``, `path: source.directory`, `fileLocation: source.directory`, `sourceKind: "builtin"`, `sourceLabel: "Built in"`, `projectLinks: []`. Two notes: (a) `source.directory` is presumably `"builtin://skills/<name>"` (per the test fixture at `skills.test.ts:160-162`), which is a synthetic URI not a real filesystem path — the UI will need to handle this in any "open in file manager" affordance (verified at the SkillDetailPage chunk: `isBuiltin` gates the edit/open buttons off). (b) `projectLinks: []` is correct — built-ins can't be linked to projects since they aren't on-disk overlays.
- `ui/goose2/src/features/skills/api/skills.ts:130-145`: parallel fetch via `Promise.all([fetchSources("skill"), fetchSources("builtinSkill").catch(() => null)])`. The `.catch(() => null)` on builtin is the load-bearing back-compat behavior — older goose-server versions that don't recognize `builtinSkill` as a source type will reject the call, and the UI must keep working. The downstream consumer at line 264 conditionally spreads the builtin response (`...(builtinResponse ? [{...}] : [])`), correctly skipping the slot. Pinned by the test at `skills.test.ts:91+` ("keeps filesystem skills when built-in skill listing fails").
- `ui/goose2/src/features/skills/api/skills.test.ts:30+` and `:138+`: new test fixtures inject the builtin-listing call as the *second* `mockResolvedValueOnce`. The strict ordinal (1: skill global, 2: builtinSkill, 3..N: per-project skill) is fragile to a refactor that switches to `Promise.all` ordering — currently it works because `Promise.all` preserves index order, but if `Promise.allSettled` were swapped in or the helper signatures change, the mock chain would silently desync. Worth switching to a `mockImplementation` keyed on the call args rather than ordinal.
- `ui/goose2/src/features/skills/ui/SkillsToolbar.tsx:656-674`: new "Built in" filter chip, gated on `skills.some((skill) => skill.sourceKind === "builtin")` — the chip only appears when there's at least one builtin skill in the listing. Good — older servers that fail the `builtinSkill` query won't render an empty chip.
- `ui/goose2/src/features/skills/ui/SkillsView.tsx:374-395`: the "Built in" group section is appended *after* the global and project sections, separated by group `id: "builtin"`. Conditional rendering (`...(builtinSkills.length > 0 ? [...] : [])`) keeps the header out when there are no builtins.
- `ui/goose2/src/features/skills/ui/SkillDetailPage.tsx:405-566`: `isBuiltin = skill.sourceKind === "builtin"` gates: edit-instructions affordance hidden (`{!isBuiltin ? ... : null}` at line 416), open-in-finder/explorer hidden (line 566 the inverse case shows a "Built in" badge instead). Read-only invariant looks well-enforced at the UI seam.
- `ui/goose2/src/shared/i18n/locales/en/skills.json` and the matching `es/skills.json` add the `builtinTitle` string. The other locales presumably need updates in follow-ups; not in this diff.

## Verdict

**merge-after-nits**

## Rationale

Solid UI integration of the upstream builtin-skills source type. The feature flag is implicit (presence of `builtinSkill` listing capability on the server) rather than explicit, and the back-compat path via `.catch(() => null)` is exactly right for a UI that ships independently of the server. Three nits: (1) the test mock pattern at `skills.test.ts` relies on ordinal `mockResolvedValueOnce` ordering; switch to a `mockImplementation((args) => ...)` keyed on `args.type` and `args.projectDir` to make the tests robust against parallelization or call-order refactors. (2) Only English and Spanish locale files are updated; the other goose2-supported locales will fall back to whatever default i18n missing-key behavior the project uses — usually OK but worth a TODO comment or a follow-up issue. (3) The synthetic `builtin://skills/<name>` URI is treated as a path in `path`/`fileLocation` — fine for the read-only UI today, but if a future affordance ever tries to `fs.readFile()` that string we get a confusing failure; consider a typed `SkillLocation = { kind: "fs"; path: string } | { kind: "builtin"; uri: string }`. None block merge.
