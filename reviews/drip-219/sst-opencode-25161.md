---
pr-url: https://github.com/sst/opencode/pull/25161
sha: fc470d17ee52
verdict: merge-as-is
---

# Remove covered workspace websocket todo

Pure cleanup: deletes a `test.todo("proxies remote workspace websocket through real Effect listener", () => {})` placeholder at `packages/opencode/test/server/httpapi-workspace.test.ts:136` and drops the now-unused `test` symbol from the `bun:test` import on line 1. Eight lines net — `-1 -2` in test list, `-1` from the import — and nothing else moves. The justification is implicit in the diff and explicit in the PR title: the websocket-proxy path the todo was reserving is now exercised by other tests in the same describe block (e.g. `it.live("serves read endpoints"...)` at `:138` and the workspace-routing tests added by the prior #25152 / #25139 wave that landed in drip-217), so the placeholder is dead context.

`test.todo` placeholders are load-bearing in two narrow ways: (1) they keep a name reserved so a future implementer doesn't pick a colliding one, and (2) they show up in the runner's "skipped" count, generating gentle pressure to actually write them. Once the underlying coverage exists elsewhere, both reasons evaporate and leaving the todo behind only trains reviewers to ignore "skipped" counts — which is exactly the failure mode you don't want when an actually-skipped test slips in. Surgical cleanup at the right granularity.

## what I learned
A `test.todo` is a debt receipt; pay it (delete it) once the work is done elsewhere, don't roll it forward as decoration. Same discipline applies to `// TODO:` comments next to code that already does the thing — leaving them there inverts the signal-to-noise ratio of every future grep for `TODO`.
