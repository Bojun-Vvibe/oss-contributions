# sst/opencode #25272 — refactor: rename workspace adapters

- Link: https://github.com/sst/opencode/pull/25272
- Head SHA: `405d82d095643020f007f92775634a4313463557`
- Author: kitlangton

## Summary
Pure rename of "adaptor" → "adapter" across the workspace control-plane: directory `src/control-plane/adaptors/` → `src/control-plane/adapters/`, type `Adaptor` → `Adapter`, plugin types `WorkspaceAdaptor` → `WorkspaceAdapter` and registration helper `registerAdaptor` → `registerAdapter`, REST surface `/experimental/workspace/adaptor` → `/experimental/workspace/adapter`, regenerated `packages/sdk/openapi.json` and `packages/sdk/js/src/v2/gen/{sdk,types}.gen.ts`, plus mirror updates in two specs and seven test files. Diff is a clean +225/-225 with one-for-one substitutions and no behavior change. Marked as replacement for #25025.

## Line-level observations
- `packages/opencode/src/cli/cmd/tui/component/dialog-workspace-create.tsx:13,108-130` — type alias `Adaptor` becomes `Adapter`, signal `adaptors`/`setAdaptors` renamed, fetch URL at `:114` flips to `/experimental/workspace/adapter`, and the toast text at `:118` updated to "Failed to load workspace adapters". This is the single user-visible string change in the PR; no missed call sites in this component.
- `packages/opencode/specs/effect/http-api.md:201` and `:293` — the bridge-parity table row for `workspace` has its capability list rewritten from `adaptor/list/...` to `adapter/list/...` and the explicit `GET /experimental/workspace/adaptor` checklist entry is renamed. Both spec docs (`http-api.md` and `schema.md:356`) plus the source path ledger are kept in lockstep, which is the right discipline for a cross-cutting rename — the next contributor reading the spec won't see drift.
- `packages/opencode/src/server/routes/instance/httpapi/groups/workspace.ts` plus the corresponding handler file — route group definition and handler symbol both renamed. Because the OpenAPI snapshot at `packages/sdk/openapi.json` is regenerated and the SDK `.gen.ts` files are committed, downstream JS clients pick up the new path through normal codegen rather than runtime probing.
- `packages/opencode/test/control-plane/adapters.test.ts` (renamed from `adaptors.test.ts`) and `test/plugin/workspace-adapter.test.ts` exercise the new symbol names; the pre-existing `httpapi-workspace.test.ts` and `httpapi-workspace-routing.test.ts` are updated to hit the new path. No old-name compatibility shim is left behind, which is consistent with the project's stance on experimental routes (`/experimental/*` is explicitly unstable) but worth a release-note callout for fork users.
- No deprecation alias for the prior `/experimental/workspace/adaptor` path or for the `WorkspaceAdaptor` plugin type. Any downstream plugin pinned to the prior name will break on version bump. Given the `/experimental/` prefix this is in-policy, but a one-line CHANGELOG/migration note in the PR body (rather than only "Replaces #25025") would help fork maintainers.

## Verdict
`merge-as-is` — mechanical rename done with full breadth (source, tests, two spec docs, generated SDK, OpenAPI schema, plugin surface) and zero behavior delta. The +225/-225 symmetry and the typecheck-and-test commands listed in the PR body are exactly what this kind of cleanup needs. The only nit (a migration note) is a documentation polish, not a blocker.
