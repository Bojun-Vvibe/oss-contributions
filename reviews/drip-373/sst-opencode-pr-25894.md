# sst/opencode #25894 — fix(core): use current workspace with /new; fix warping into local project

- **Head:** `760a216` (`760a2163fba26919977a8a6498560b234590f236`)
- **Repo:** sst/opencode
- **Scope:** `dialog-workspace-create.tsx`, `prompt/index.tsx`, `server/routes/instance/httpapi/public.ts`, generated SDK (`v2/gen/sdk.gen.ts`, `v2/gen/types.gen.ts`)

## What the diff does

Three logically-distinct fixes bundled:

1. **`dialog-workspace-create.tsx:129-136`** — workspace recents listing was first slicing to 3 then filtering, so disconnected workspaces consumed slots in the recents panel and the user saw fewer than 3 actionable workspaces. Diff swaps order: filter to `status() === "connected"` first, then `slice(0, 3)`. Correct fix.

2. **`prompt/index.tsx:185, 865, 869, 1149-1158`** — when `/new` runs in a session that doesn't yet have a workspace selected and the user hasn't picked one, fall back to `project.workspace.current()` instead of sending `workspace: undefined` (which the server interprets as "no workspace"). New `defaultWorkspaceID = createMemo(() => props.workspaceID ?? project.workspace.current())` at `:185` is the load-bearing change. The `iife` at `:864-870` now resolves to `defaultWorkspaceID()` when nothing is selected, and the `workspaceWarpHint` memo at `:1149-1158` synthesizes a stub hint from the current workspace so the prompt UI shows the right warp target.

3. **`server/routes/instance/httpapi/public.ts:149-158` + generated SDK** — `/experimental/workspace/warp`'s OpenAPI spec for `body.id` now declares `string | null` instead of `string`, so a warp request can explicitly carry "no workspace" intent. The legacy-OpenAPI shim at `:149-158` mirrors the existing per-property `anyOf` pattern used for `branch`/`extra` in adjacent paths.

## Concerns

1. **`/new` semantics change is observable.** Previously `/new` with no explicit workspace created a session detached from any workspace; now it inherits `project.workspace.current()`. For a user who relied on `/new` as the way to escape from a workspace context this is a behavior change. Defensible — the new behavior matches what most users expect — but worth a release note.
2. **`workspaceWarpHint` fallback at `:1149-1158`** synthesizes `workspaceType` and `status` from `project.workspace.get(workspaceID)` and `project.workspace.status(workspaceID)`. If `get()` returns `undefined` (workspace was deleted between memo trigger and resolution), the hint shows raw `workspaceID` as the name and `status: "error"`. Acceptable graceful-degrade.
3. **Generated SDK churn (`sdk.gen.ts`, `types.gen.ts`) is mechanical** and matches the OpenAPI change at `public.ts`. Only the `id?: string | null` widening at `:1022` is meaningful; reviewers can skip the rest.
4. **No new test**. The `iife` at `:864-870` and the warp-hint fallback are both branchy enough to warrant a test that pins both `props.workspaceID` set vs unset paths. Not blocking, but easy to add.

## Verdict

**merge-after-nits** — split-bundle is fine because the three changes are tightly coupled around the `/new`-into-workspace flow, but please add a release note line for the `/new` behavior change and one regression test for the new fallback in `prompt/index.tsx`. Head `760a216`.
