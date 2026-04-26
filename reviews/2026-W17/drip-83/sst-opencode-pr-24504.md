# sst/opencode PR #24504 — fix: remove unnecessary any type from catch clause in github.ts

- Repo: sst/opencode
- PR: #24504
- Head SHA: `c462a2ab`
- Author: @alfredocristofano
- Diff: targeted single-line `catch (e: any)` → `catch (e)` in `packages/opencode/src/cli/cmd/github.ts:689` (plus same 100-file Config-import-migration drag-along seen across this contributor's batch — #24502/#24503/#24507)
- Closes: #24327

## What changed

Single-token deletion: `} catch (e: any) {` becomes `} catch (e) {` at `packages/opencode/src/cli/cmd/github.ts:689`. TypeScript with `useUnknownInCatchVariables: true` (the modern default in `tsconfig` strict mode) will infer `unknown` for the omitted annotation, which is the correct type for caught values. The body of the catch already uses `e instanceof Error` and `e instanceof Process.RunFailedError` narrowing before accessing properties, so removing `any` does not require any other change.

## Specific observations

- The PR also lands the documented style change in the new `AGENTS.md`: `- Avoid \`any\`; prefer precise types.` is one of the explicit ground rules. So this edit is the project's own style enforcing on existing code — exactly the right kind of cleanup.
- One worth-checking detail: this catch block's body assigns `msg = e` and later uses `msg` as if it were a string-y thing in some branches (visible in the surrounding context). With `e: unknown`, downstream uses of `msg` will need to also be type-narrowed. From the snippet shown, `console.error(e instanceof Error ? e.message : String(e))` is fine, but the trailing `let msg = e` followed by other manipulation should be skimmed to confirm no implicit-any-via-assignment regression. If `tsc` passes locally and CI is green, the worry is moot.
- Same diff-hygiene caveat as the rest of this contributor's batch in this round: PR carries the 100-file `Config` import-path migration (`"../config"` → `"../config/config"`) plus AGENTS.md rewrite that has nothing to do with catch-clause typing. Maintainers should request the contributor to split: one PR for `import` migration, one PR for AGENTS.md, and rapid-fire single-file PRs for each `any` cleanup. Reviewing 100 files for a one-token change is hostile to reviewer time.
- Pair with #24503 (mcp catch unknown handling) and #24502 (workspace-restore log) — same author, same theme, same drag-along noise. They should ideally be served as one batch with a clean rebase, or as four micro-PRs. The current shape is the worst of both worlds.

## Verdict

`merge-after-nits`

## Rationale

Substantively correct one-token cleanup that aligns existing code with the project's own AGENTS.md style policy. Diff-hygiene nit — the 99 unrelated files dragged in make this look like a config-migration PR, not a type-cleanup PR. Either rebase clean or batch with the other catch-cleanup PRs.
