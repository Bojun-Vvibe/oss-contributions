# sst/opencode#25797 — Fix workspace warp SDK null id

- Head SHA: `047fdd65f296672937cc03f82f3994b8c8434002`
- Author: Hona
- Verdict: **merge-as-is**

## Summary

The legacy OpenAPI emitter for `/experimental/workspace/warp` was
publishing a non-nullable `id` on the request body, but the actual
runtime semantics (and the use case driving this fix —
`workspace.warp({ id: null })` to detach to a local-project context)
require nullable. The fix patches the emitter and regenerates the JS SDK
types so callers typecheck.

## Specific references

- `packages/opencode/src/server/routes/instance/httpapi/public.ts:149-152`
  adds a focused branch in `matchLegacyOpenApi` mirroring the existing
  `branch`/`extra` nullable-property carve-outs:
  ```ts
  if (path === "/experimental/workspace/warp" && method === "post") {
    const properties = operation.requestBody.content?.["application/json"]?.schema?.properties
    if (properties?.id) properties.id = nullable(properties.id)
  }
  ```
- `packages/sdk/js/src/v2/gen/sdk.gen.ts:1022` — generated parameter
  type flips from `id?: string` to `id?: string | null`.
- `packages/sdk/js/src/v2/gen/types.gen.ts:6668` —
  `ExperimentalWorkspaceWarpData.body.id` now `string | null`.

## Reasoning

This is a contract-fidelity fix: the SDK now matches what the route has
always accepted. Surface area is minimal (one new branch in the
emitter, two regenerated lines in the SDK). No runtime behavior change
on the server.

The mechanism — listing endpoints by `(path, method)` tuples in
`matchLegacyOpenApi` — is itself a known anti-pattern (you have to
remember to add a carve-out every time a route grows a nullable field),
but that's pre-existing tech debt and not this PR's job. Ship.
