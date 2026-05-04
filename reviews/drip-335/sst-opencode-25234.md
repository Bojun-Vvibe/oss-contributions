# Review: sst/opencode #25234

- **Title:** docs: update SDK docs to reference v2 SDK exclusively
- **Head SHA:** `020b2240c245ba954d993a6a176a91f5115d05d9`
- **Size:** +7 / -7
- **Files touched:**
  - `packages/web/src/content/docs/sdk.mdx`

## Critique

Pure docs change. Updates `sdk.mdx` to import from `@opencode-ai/sdk/v2` instead of the bare `@opencode-ai/sdk` entrypoint, and rewrites the `typesUrl` to point at `packages/sdk/js/src/v2/gen/types.gen.ts`.

Specific lines:

- `sdk.mdx:9` — `typesUrl` now points at `v2/gen/types.gen.ts`. Verify that path actually exists on `dev` branch; if the SDK package still publishes both `v1` and `v2` paths, broken links here will be the most user-visible regression.
- `sdk.mdx:18, 27, 36, 45` — all four import lines changed from `"@opencode-ai/sdk"` to `"@opencode-ai/sdk/v2"`. Consistent.
- `sdk.mdx:55` — API method renamed in the docs from `postSessionByIdPermissionsByPermissionId` to `permission.reply`. This is a docs-only rename; the underlying SDK must already expose `permission.reply` for this doc to be accurate. Worth a quick grep against the v2 SDK package before merging.
- `sdk.mdx:64` — `file.read` return type changed from `{ type: "raw" | "patch", content: string }` to `{ type: "text" | "binary", content: string }`. This is a meaningful API contract change disclosed only here; if upstream code paths still emit `"raw"` / `"patch"`, doc and code will be misaligned. Cross-check with the actual v2 type definition.

Risk: low (docs only). The file.read return-shape claim is the highest-leverage thing to validate against generated types before merge — if it's wrong, downstream consumers will hit type errors.

No banned strings, no security implications.

## Verdict

`merge-after-nits`
