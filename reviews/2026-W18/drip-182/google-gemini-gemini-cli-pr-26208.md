---
pr: google-gemini/gemini-cli#26208
sha: baeccee504acbf3d57aac0836ce3d2720f83c107
verdict: merge-after-nits
reviewed_at: 2026-04-29T18:31:00Z
---

# fix: suppress duplicate extension warnings during startup

## Context

`gemini.tsx` calls `loadCliConfig` twice during startup — once for
`partialConfig` (auth/sandbox bootstrap) and once for the full
interactive session. Both calls were unconditionally instantiating
`new ExtensionManager(...)` and calling `loadExtensions()`, which
emits warnings (missing settings, MCP deprecations) through
`coreEvents`. Result: every warning bubble showed twice. Fix in
`packages/cli/src/config/config.ts` adds a `skipExtensions?: boolean`
to `LoadCliConfigOptions` and the bootstrap call passes `true`.

## What's good

- Targeted, minimal change. Only the bootstrap call is opted out
  (`packages/cli/src/gemini.tsx` line 410: `skipExtensions: true`),
  and the second/interactive call still loads extensions normally.
- The `SimpleExtensionLoader([])` fallback at config.ts line 681
  (`const finalExtensionLoader = extensionManager ?? new SimpleExtensionLoader([])`)
  preserves the type contract for downstream consumers
  (`loadServerHierarchicalMemory`, the final `Config` object's
  `extensionLoader` field). Good defensive choice — avoids forcing
  every callsite to handle `undefined`.
- Optional chaining on `extensionManager?.getExtensions()?.find(...)`
  for `extensionPlanSettings` is correct: if extensions are skipped,
  there are no plan settings to find.

## Concerns / nits

1. **`pr_body.md` checked in.** The diff includes a top-level
   `pr_body.md` file that's just the PR description duplicated.
   That's almost certainly an accident from a `gh pr create
   --body-file pr_body.md` workflow. Should be removed before merge
   (or `.gitignore`d).
2. **`skipExtensions` semantics aren't tested in this diff.** The PR
   description references a manual test file path
   (`packages/cli/src/config/skipExtensions.test.ts`) but the diff
   shown doesn't contain it. Either add the test or update the PR
   body — saying "added/updated tests" is checked in the checklist
   but the test isn't visible.
3. **Subtle behavior change.** When `skipExtensions: true`,
   `extensionPlanSettings` is `undefined`, so any setting that
   *only* comes from an extension plan (e.g. plan-injected
   `includeDirectories`) is silently absent during the bootstrap
   pass. If anything in the auth/sandbox bootstrap actually consults
   that, this becomes a regression. Worth a quick audit of what
   `partialConfig` is used for between those two `loadCliConfig`
   calls.

## Verdict

`merge-after-nits` — remove the stray `pr_body.md`, add or point to
the actual test, and confirm bootstrap-phase consumers don't depend
on extension plan settings.
