# sst/opencode PR #24503 — fix: replace catch (e: any) with proper unknown handling in mcp/index.ts

- Repo: sst/opencode
- PR: #24503
- Head SHA: `a5e7081e`
- Author: @alfredocristofano
- Diff: +897/-789 across 100 files

## What changed

The PR title and description scope the change to a single file (`packages/opencode/src/mcp/index.ts`) — replacing two `catch (e: any) { ... return e }` blocks with `instanceof Error`-narrowed handlers and adding logging to a previously silent `process.kill` `catch {}` block. The actual diff, however, includes ~98 unrelated files: a full `AGENTS.md` rewrite, a `Config` import-path migration (`"../config"` → `"../config/config"`) propagated across roughly 30 files (`agent/agent.ts`, `auth/index.ts`, every `cli/cmd/*.ts`, `node.ts`, `server/server.ts`, etc.), and `infra/app.ts` / schema regen.

## Specific observations

- The targeted `mcp/index.ts` change is genuinely clean: two `Effect.tryPromise.catch` blocks at `packages/opencode/src/mcp/index.ts:716-723` and `:741-748` correctly narrow `e: any` → `e: unknown`, derive `msg = e instanceof Error ? e.message : String(e)`, and return `e instanceof Error ? e : new Error(msg)` so downstream `Effect.orElseSucceed` / `Effect.map` consumers always get a real `Error`. The previously silent `process.kill(dpid, "SIGTERM")` `catch {}` at `:730-733` is upgraded to `log.debug("process.kill SIGTERM failed", { error: e, pid: dpid })` which is the right call for diagnosing zombie-MCP cleanup.
- The PR is severely overscoped relative to its title. The 100-file `Config` import path rename (`"../config"` → `"../config/config"`) is a workspace-wide refactor that will conflict with every in-flight branch, breaks `git blame` on those import lines for an unrelated reason, and has no entry in the PR body. If `packages/opencode/src/config/index.ts` was deleted in favor of explicit `config/config` imports, that needs its own PR + CHANGELOG / breaking-change note (downstream `@opencode-ai/sdk` re-exports at `packages/opencode/src/node.ts:1` are also affected: `export { Config } from "./config/config"`).
- The `AGENTS.md` replacement at lines 1-144 swaps the project's existing concise style guide ("Avoid `try`/`catch` where possible", "Avoid using the `any` type") for an entirely different "OpenCode AGENTS.md" doc with monorepo map, dev commands, and PR conventions. This is a governance/process change and absolutely does not belong in a one-file mcp typing fix.

## Verdict

`request-changes`

## Rationale

The actual `mcp/index.ts` typing fix is good and would land cleanly on its own, but bundling a workspace-wide `Config` import-path migration plus a full `AGENTS.md` rewrite under a "fix mcp typing" title is unreviewable and a future-blame disaster. Split into three PRs: (1) the `mcp/index.ts` two-block typing fix as titled; (2) the `Config` re-export move with a clear rationale; (3) the `AGENTS.md` rewrite as a `docs:` PR with maintainer sign-off.
