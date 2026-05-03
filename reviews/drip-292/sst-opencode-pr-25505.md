# sst/opencode PR #25505 — Fix plugin agent registration issue

- **Repo:** sst/opencode
- **PR:** #25505
- **Head SHA:** `9911f311421ba5e112b8c32351b564cb2af162b9`
- **Author:** kill74
- **Title:** Fix plugin agent registration issue
- **Diff size:** +19 / -8 across 2 files
- **Drip:** drip-292

## Files changed

- `.opencode/opencode.jsonc` (+2/-1) — adds `"plugin": ["oh-my-opencode@3.17.12"]` and a stray missing-newline at EOF.
- `packages/opencode/src/plugin/shared.ts` (+17/-7) — introduces a `PLUGIN_PACKAGE_RENAMES` map (`oh-my-openagent` → `oh-my-opencode`) and threads renames through both `parse()` and `resolvePluginTarget()`.

## Specific observations

- `shared.ts:14-16` — defines `PLUGIN_PACKAGE_RENAMES: Record<string, string> = { "oh-my-openagent": "oh-my-opencode" }`. Fine as a single-key shim but should land with a comment naming the upstream rename and a deprecation horizon, otherwise the next maintainer will have no idea why this exists.
- `shared.ts:21-23` (inside `parse`) — `const renamed = PLUGIN_PACKAGE_RENAMES[spec] || spec` only matches the *exact* spec string. A user who pins `oh-my-openagent@2.4.0` (any non-bare version) will silently bypass the rename because the map key is the bare package name. Either parse first then look up `parsed.name`, or strip version before lookup.
- `shared.ts:55-61` (inside `resolvePluginTarget`) — calls `parsePluginSpecifier(spec)` and then re-runs `parse(next)`. The re-parse path is correct but `parsePluginSpecifier` is not visible in this diff hunk; reviewer should confirm it returns `{ pkg, version }` shape consistent with what npa would yield (lines 60-61 assume `parsed.pkg` and `parsed.version`).
- `.opencode/opencode.jsonc:6` — pinning `oh-my-opencode@3.17.12` in the repo's own dogfood config is reasonable, but the trailing newline removal at line 17-18 is gratuitous diff noise; ask author to restore the final `\n`.
- The import block reordering at `shared.ts:1-7` is purely cosmetic and unrelated to the bug fix; not blocking but bloats the diff.

## Verdict: `merge-after-nits`

The core rename-shim is the right fix shape for #25441, but the lookup is keyed on the raw spec rather than the parsed package name, which makes it brittle the moment any user includes a version, tag, or registry prefix. Tightening the lookup to operate on the parsed package name (and reverting the EOF newline removal) would land this cleanly.
