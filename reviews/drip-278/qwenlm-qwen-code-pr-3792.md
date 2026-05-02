# Review — QwenLM/qwen-code#3792

- PR: https://github.com/QwenLM/qwen-code/pull/3792
- Title: fix(core): address post-merge monitor tool and UI routing issues
- Head SHA: `9d25aaf155c49509f0fdcfa3c117483d244342b2`
- Size: +199 / −199 across 14 files
- Verdict: **merge-after-nits**

## Summary

Post-merge cleanup of a monitor-tool / background-task feature.
Three substantive things:
1. Extracts the duplicated `hasBlockingBackgroundWork` and
   `resetBackgroundStateForSessionSwitch` helpers from
   `clearCommand.ts` and `useResumeCommand.ts` into a new
   `packages/cli/src/ui/utils/backgroundWorkUtils.ts`.
2. Tweaks the monitor tool and its registry tests.
3. Re-routes the monitor tool-call rendering in both the VS Code
   companion webview and the standalone webui through a shared
   routing module.

## Evidence

- `packages/cli/src/ui/utils/backgroundWorkUtils.ts` (new) — single
  source of truth for the two helpers; both consumers now
  `import { hasBlockingBackgroundWork,
  resetBackgroundStateForSessionSwitch } from
  '../utils/backgroundWorkUtils.js'`.
- `packages/cli/src/ui/commands/clearCommand.ts` — removes the
  inline `hasBlockingBackgroundWork(config: Config): boolean` and
  `resetBackgroundStateForSessionSwitch(config: Config): void`
  definitions verbatim, swaps to the new import, drops the now-
  unused `type Config` import.
- `packages/cli/src/ui/hooks/useResumeCommand.ts` — same dedup as
  above; the previously-duplicated functions are gone.
- `packages/core/src/permissions/permission-manager.ts` and
  `permissions/rule-parser.ts` — touched as part of the same PR
  (changes not inspected line-by-line here, but the file list
  suggests a related permission-rule refactor that the monitor tool
  depends on).
- `packages/core/src/services/monitorRegistry.test.ts` and
  `packages/core/src/tools/monitor.test.ts` plus `monitor.ts` —
  test+impl updates for the monitor tool.
- `packages/vscode-ide-companion/src/webview/components/messages/toolcalls/index.tsx`
  and `packages/webui/src/components/toolcalls/{index.ts,routing.ts}`
  — shared routing module so both webviews dispatch monitor
  tool-calls through the same code path.

## Notes / nits

- Diff is exactly +199/−199, which screams "pure move/refactor". But
  it spans `cli`, `core`, `vscode-ide-companion`, and `webui`
  packages — please confirm in the PR body that each package's
  build + test suite was run independently (a workspace-wide test
  pass can mask a package boundary regression if the affected
  package has a stale build artifact).
- `backgroundWorkUtils.ts` only contains the two helpers; consider
  also moving any related `Config`-poking helpers (e.g. the
  background-shell predicate, if it exists separately) here in a
  follow-up to prevent the dedup from regressing.
- The rule-parser / permission-manager changes are not obviously
  related to "monitor tool and UI routing" from the title — splitting
  them into a separate PR would have made review faster. Not a
  blocker for this one if they're genuinely a single logical
  rollback / fixup.

Mechanical refactor + small fixups. Ship after a quick sanity check
that the cross-package builds are all green.
