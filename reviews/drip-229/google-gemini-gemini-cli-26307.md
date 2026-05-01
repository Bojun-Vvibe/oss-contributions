# google-gemini/gemini-cli #26307 — feat(config): enable Gemma 4 models by default via Gemini API

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26307
- **Head SHA**: `620123339626`
- **Files reviewed**: `.gemini/settings.json`, `docs/cli/settings.md`, `docs/reference/configuration.md`, `packages/cli/src/config/settingsSchema.ts`, `packages/core/src/config/config.ts`, `packages/core/src/config/config.test.ts`, `packages/core/src/config/models.ts`, `packages/core/src/config/models.test.ts`, `schemas/settings.schema.json`

- **Date**: 2026-05-01 (drip-229)

## Context

Promotes `experimental.gemma` from opt-in (`false` default) to
opt-out (`true` default), removes the "experimental" label in the
docs, and adjusts the description to clarify Gemma 4 access is via
the Gemini API. Also drops the now-redundant explicit
`"gemma": true` override from the repo's own `.gemini/settings.json`.

## Diff (9 files, +15 -16)

`packages/core/src/config/config.ts:1182`:

```diff
-    this.experimentalGemma = params.experimentalGemma ?? false;
+    this.experimentalGemma = params.experimentalGemma ?? true;
```

`packages/core/src/config/models.ts:458`:

```diff
-  experimentalGemma: boolean = false,
+  experimentalGemma: boolean = true,
```

`packages/cli/src/config/settingsSchema.ts:2057-2061`:

```diff
-        default: false,
-        description: 'Enable access to Gemma 4 models (experimental).',
+        default: true,
+        description: 'Enable access to Gemma 4 models via Gemini API.',
```

Plus matching mirror updates in `schemas/settings.schema.json:3049-3057`,
`docs/cli/settings.md:166`, `docs/reference/configuration.md:1761-1764`,
and the `.gemini/settings.json:6` line removed.

Tests at `config.test.ts:3676,3686` and `models.test.ts:598-600` are
flipped to assert the new defaults (returns `true` when
`experimentalGemma` is not provided; `isActiveModel(GEMMA_4_*)` returns
`true` by default).

## Observations

1. **Default flipped at every source-of-truth.** The default exists at
   four canonically-distinct sites: (a) the `Config` constructor's
   `??` fallback in `config.ts`, (b) the function-argument default in
   `models.ts:isActiveModel`, (c) the settings-schema metadata
   `default:` in `settingsSchema.ts`, and (d) the JSON-schema mirror
   in `schemas/settings.schema.json`. Plus the user-facing prose in
   the two docs files. All five are flipped consistently — the
   reviewer can scan the diff and confirm there is no remaining
   `false` fallback that would shadow the new default at one entry
   point but not another. That's the right shape for a default flip.

2. **Test assertions correctly flipped.** `config.test.ts:3686`
   asserts `getExperimentalGemma()` returns `true` when no override
   given; `models.test.ts:598-600` asserts the two Gemma 4 model IDs
   are now active by default. The earlier "should return false when
   not provided" arm at `:3673` is preserved (now reading "should
   return false when explicitly set to false") which is the correct
   anti-behavior pin — confirms the explicit-`false` override path
   still works.

3. **Repo-config cleanup is correct.** Removing
   `"gemma": true` from `.gemini/settings.json` after the default
   becomes `true` is the right hygiene — keeps the file as a record
   of explicit deviations from defaults rather than a record of
   defaults.

## Nits

- **Description loses the "experimental" signal entirely.** The new
  copy is "Enable access to Gemma 4 models via Gemini API." but the
  setting still lives under the `experimental` namespace
  (`experimental.gemma`), is still annotated `category: 'Experimental'`
  in the schema, and `requiresRestart: true`. Users reading the
  setting label "Gemma Models" with description "Enable access to
  Gemma 4 models via Gemini API" might reasonably expect this is
  now stable; if it is in fact still experimental (as the namespace
  implies) the description should say so. Or, conversely, if it's
  considered stable enough to default-on, consider lifting it out of
  the `experimental.*` namespace in a follow-up.

- **No release-notes / migration entry visible in this diff.** Users
  who explicitly disabled `gemma` before will still have their
  setting honored (`?? true` only fills the absent case), but users
  who never set it will see new model entries in their picker after
  upgrade. Worth a one-line CHANGELOG note.

- **`isActiveModel(...)` has six positional boolean params now.**
  The default flip at `models.ts:458` is the third change to defaults
  on this signature. Worth a refactor to a single options object
  before the next flip lands and somebody passes `false` in the
  wrong slot.

## Verdict

`merge-after-nits` — clean default flip with all five source-of-truth
sites updated consistently and tests correctly inverted with the
explicit-override anti-behavior arm preserved. Nits are the
description's lost "experimental" signal, missing migration note,
and the long-overdue options-object refactor of `isActiveModel`.
