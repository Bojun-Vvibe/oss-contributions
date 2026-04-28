# block/goose#8797 — feature: handle removed extensions

- **Repo:** [block/goose](https://github.com/block/goose)
- **PR:** [#8797](https://github.com/block/goose/pull/8797)
- **Head SHA:** `21b45068b34d5ec5504f6bbd8cd327fb6180bc08`
- **Size:** +150 / -178 across 6 files (UI desktop only)
- **State:** MERGED

## Context

Two related correctness fixes in goose's desktop UI:

1. **Session extensions weren't shown in the bottom-menu picker** when the
   user had removed those extensions from global config. This left users in
   a state where the *session* still had extensions wired up but they had
   no UI to see or disable them.
2. **`syncBundledExtensions` could only create/update bundled extensions,
   never remove deprecated ones.** Previously the only deprecation
   mechanism was the hardcoded `isDeprecatedGoogleDriveExtension` predicate
   inside `bundled-extensions.ts`. That doesn't scale — every future
   deprecation needed code changes — and conflated "is this the gdrive
   bundle" with "is this deprecated".

## Design analysis

### 1. Bundled-extension prune via JSON allowlist (`bundled-extensions.ts`)

The refactor moves from "hardcoded `isDeprecatedGoogleDriveExtension`
predicate + special-case branching inside `syncBundledExtensions`" to
"separate `pruneDeprecatedBundledExtensions` step driven by
`deprecated-bundled-extensions.json`":

```ts
// bundled-extensions.ts:33-58 (new)
export async function pruneDeprecatedBundledExtensions(
  existingExtensions: FixedExtensionEntry[],
  removeExtensionFn: (id: string) => Promise<void>,
): Promise<FixedExtensionEntry[]> {
  const deprecatedExtensionIds = new Set(getDeprecatedBundledExtensions().map((ext) => ext.id));
  const remainingExtensions: FixedExtensionEntry[] = [];

  for (const existingExt of existingExtensions) {
    if (!isBundledExtension(existingExt)) {
      remainingExtensions.push(existingExt);
      continue;
    }
    if (!deprecatedExtensionIds.has(nameToKey(existingExt.name))) {
      remainingExtensions.push(existingExt);
      continue;
    }
    await removeExtensionFn(nameToKey(existingExt.name));
  }
  return remainingExtensions;
}
```

This is a clean three-state filter:

- **Not bundled** → keep (user-installed extensions are never auto-pruned).
- **Bundled but not in deprecated list** → keep.
- **Bundled and deprecated** → call `removeExtensionFn`, drop from the
  returned list.

The returned list is then passed to `syncBundledExtensions` to re-add any
non-deprecated bundles. The orchestration is at
`ConfigContext.tsx:230-234`:

```ts
const removeExtensionForSync = async (name: string) => {
  await apiRemoveExtension({ path: { name } });
};
extensions = await pruneDeprecatedBundledExtensions(extensions, removeExtensionForSync);
await syncBundledExtensions(extensions, addExtensionForSync);
```

That ordering is correct: prune first, then sync. If sync ran first, a
deprecated bundle that needed re-adding under a *different* id (the
"replace gdrive with skills" use case the PR cites) would never be
processed because sync's `existingExt && isBundledExtension(existingExt)`
short-circuit would skip it.

The new test `'allows same-id bundled extensions to be re-added after
prune'` (in the test file diff) validates exactly this round-trip: prune
removes `googledrive`, then `syncBundledExtensions` re-adds it from the
JSON manifest. Strong test.

### 2. Session-extension visibility fix (`BottomMenuExtensionSelection.tsx:234-258`)

The new merge logic:

```tsx
const sessionExtensionKeys = new Set(sessionExtensions.map((ext) => nameToKey(ext.name)));
const globalExtensionKeys  = new Set(allExtensions.map((ext) => nameToKey(ext.name)));

const mergedExtensions = allExtensions.map((ext) => ({
  ...ext,
  enabled: sessionExtensionKeys.has(nameToKey(ext.name)),
}) as FixedExtensionEntry);

for (const sessionExtension of sessionExtensions) {
  if (globalExtensionKeys.has(nameToKey(sessionExtension.name))) continue;
  mergedExtensions.push({ ...sessionExtension, enabled: true });
}

return mergedExtensions;
```

Three things this gets right:

1. **`nameToKey` everywhere.** The previous code compared `ext.name`
   directly, which would fail for any extension whose display name
   differs from its key (e.g., `"Google Drive"` vs `"googledrive"`). The
   set-based lookup with `nameToKey` is the right normalization.
2. **Loop over session extensions to surface orphans.** Any session-only
   extension (one in `sessionExtensions` but not in `allExtensions`) gets
   appended with `enabled: true`. The user can now see and toggle it.
3. **Discriminator on `globalExtensionKeys`, not on the merged-list
   contents.** Avoids the O(n²) "did I already add this?" check by
   precomputing the set once.

### 3. Test refactor (`bundled-extensions.test.ts`: -121 / +68)

Net deletion of ~50 lines because the `isDeprecatedGoogleDriveExtension`
predicate (and its 4-5 unit tests) is gone — replaced by the more general
`pruneDeprecatedBundledExtensions` flow tested as an integration. The new
tests cover:

- "removes deprecated bundled extensions"
- "does not remove non-bundled extensions" (negative case)
- "allows same-id bundled extensions to be re-added after prune" (the
  round-trip case)

The deletion is appropriate — the per-predicate tests don't add coverage
that the integration tests don't already give, and removing them prevents
the test file from growing every time a new deprecation lands.

## Risks / nits

1. **`deprecated-bundled-extensions.json` ships as `[]`.** The mechanism
   is in place but no extensions are actually deprecated yet. That's fine
   for a follow-up to populate, but the empty-array initial state means
   the prune step is a no-op until someone updates the JSON. A comment in
   the JSON file (`// add { "id": "..." } entries here when removing a
   bundled extension`) would help future maintainers find the right place.
2. **`pruneDeprecatedBundledExtensions` doesn't surface remove failures.**
   If `removeExtensionFn` rejects for one extension, the loop aborts on
   the first error and the remaining extensions are never processed (and
   the partial-prune state is what gets returned). For a deprecation
   workflow this is probably correct (let the user retry on next launch),
   but the PR could log the error path explicitly so a stuck-deprecation
   bug would be diagnosable from console output alone.
3. **`BottomMenuExtensionSelection` orphan loop pushes session-only
   extensions with `enabled: true` unconditionally.** That matches the
   intent ("if it's wired to the session, it's running"), but if the
   session extension is in a half-failed state (registered with the
   session but not actually started), the toggle will show "on" while the
   tool is unavailable. Out of scope here, but worth a follow-up issue.
4. **`shouldHideTrigger` extracted as a const at line 285.** This is the
   only place the long ternary was used; pulling it out is a readability
   win. No risk.

## Verdict

**merge-after-nits.** (PR is already MERGED, so these are post-merge
follow-ups.) The architecture move from hardcoded predicate to JSON-driven
allowlist is the right call, the prune-then-sync ordering is correct, and
the orphan-session-extension fix uses `nameToKey` consistently. The two
follow-up items (JSON file comment, log on prune failure) are
documentation/observability extensions.

## What I learned

- "Hardcoded predicate per deprecated thing" is a smell. Moving to a
  JSON manifest you can update without touching code is the right
  refactor — and the test-file-shrinkage (61% deletion) is a useful
  signal that the abstraction was well-chosen.
- When two collections need to be merged and one might contain orphan
  entries the other doesn't know about, computing both sets of normalized
  keys upfront and looping the secondary collection to push orphans is
  the cleanest pattern. Avoids both N² lookups and the "did I dedupe
  this?" foot-gun.
- The prune-then-sync ordering matters: sync is idempotent only against
  the *current* desired state, so any "reset to known-good" must run
  before sync. Documenting "always prune first, then sync" near the
  orchestration site would prevent future PRs from accidentally
  reordering.
