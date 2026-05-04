# QwenLM/qwen-code#3832 — fix(sdk-python): standardize TAG_PREFIX to include v suffix

- **URL**: https://github.com/QwenLM/qwen-code/pull/3832
- **Head SHA**: `f89fb70b7adf`
- **Diffstat**: +9 / -6
- **Verdict**: `merge-as-is`

## Summary

Folds the `v` prefix into `TAG_PREFIX` itself (`sdk-python-` → `sdk-python-v`) and drops the ad-hoc `v${...}` interpolation at every call site, so the Python SDK release script produces tags like `sdk-python-v0.4.2` consistently from one source of truth.

## Findings

- `packages/sdk-python/scripts/get-release-version.js:18` — `TAG_PREFIX` is now the full prefix; this is the right normalization. Renaming the parameter `releaseTag` → `releaseVersion` in `getReleaseState` matches the new semantics (the function now receives a bare version, not a tag), so the call site at line 467 cleanly passes `versionData.releaseVersion`.
- `packages/sdk-python/scripts/get-release-version.js:283` — `const fullTag = \`${TAG_PREFIX}${releaseVersion}\`;` is the single tag-construction site. All downstream `git tag -l`, `gh release view`, and warning messages now derive from this one string — correct.
- `packages/sdk-python/scripts/get-release-version.js:514, 519` — error/warning strings dropped the spurious `v` after `TAG_PREFIX`, so messages no longer read `sdk-python-vv0.4.2`. Good catch.
- No backward-compat concern: this script generates *new* tags for *next* releases; existing `sdk-python-vX.Y.Z` tags remain valid because the new `fullTag` matches the same shape.

## Recommendation

Pure cleanup, no behavior change vs what the historical strings produced. Ship.
