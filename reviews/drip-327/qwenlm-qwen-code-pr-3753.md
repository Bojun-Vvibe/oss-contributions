# QwenLM/qwen-code #3753 — fix(cli): honor proxy setting

- SHA: `7a62b095a604908be72b6ce3528b3e343e1193f7`
- State: MERGED, +290/-12 across 8 files

## Summary

Adds first-class support for a top-level `proxy` setting in `settings.json` (alongside `--proxy` CLI flag and the existing `HTTPS_PROXY`/`HTTP_PROXY` env vars), with a documented precedence: `--proxy` > `settings.json#proxy` > env vars. New `resolveProxy(cliProxy, settingsProxy)` helper sets `undici`'s `setGlobalDispatcher(new ProxyAgent(url))` once resolved. Substantial new test file `start.test.ts` (~200 lines) exercises the precedence chain.

## Notes

- `docs/users/configuration/settings.md:73-82`: docs now correctly call out that a few compatibility settings (like `proxy`) are top-level keys rather than under a category — this is an intentional schema deviation worth flagging in the changelog.
- `packages/cli/src/commands/channel/start.test.ts:1-202`: tests are well-structured. The precedence assertions cover (a) CLI > settings > env, (b) settings > env, (c) env-only fallback. Good coverage. One missing case worth adding: when *all three* are unset, `resolveProxy()` should return `undefined` and `setGlobalDispatcher` should not be called — easy assertion to add.
- The test sets up many `vi.hoisted` mocks (~20). This is verbose but is `start.ts`'s structural shape, not the PR's fault. A `__mocks__/start-deps.ts` helper could be filed as follow-up.
- `mockChannelConnect.mockRejectedValue(new Error('stop after channel setup'))` is a clever way to short-circuit the handler after the proxy code path runs — clear comment explaining why is helpful (the diff appears to lack one).
- Already merged, so observations are informational.

## Verdict

`merge-as-is` — already merged. Follow-up suggestions: add the "all unset" precedence test; consider centralizing the channel-start mock fixtures.