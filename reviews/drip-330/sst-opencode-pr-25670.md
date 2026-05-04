# sst/opencode #25670 — fix(mcp): auto-reconnect on transport errors (#25287)

- SHA: `5c803b8db45c436e2dc1962a17ce3ce08e2b25d2`
- State: OPEN, +951/-33 across 11 files
- Closes: #25287

## Summary

Adds a transport-error classifier (`isTransportError`) and a single-flight reconnect path so MCP tool calls that hit a stale StreamableHTTP session, killed socket, or restarted server recover transparently with one retry on a fresh client. Auth errors (401/403) and ordinary tool errors are deliberately left unrouted. Ships substantial unit tests, a CI probe job, and a manual E2E supervisor script.

## Notes

- `packages/opencode/src/mcp/index.ts:607-630` — `reconnectClient` is single-flight via a `Map<string, Promise<boolean>>` keyed on the MCP name. The promise is removed in `.finally()` on both success and failure, so the map can't leak. The `getMcpConfig`+`createAndStore` chain is wrapped in `.catch()` that returns `false`; this swallows the actual error so the caller only sees "reconnect failed, rethrow original". Good for keeping business errors visible, but the warning-level log on line 619 should include the upstream stack at debug, not just `err.message`, to make support tickets traceable.
- `packages/opencode/src/mcp/index.ts:632-668` — `makeTool` replaces the file-level `convertMcpTool` (now deleted at lines ~122-150). The retry path reaches into `InstanceState.get(state)` after reconnect and pulls `next.clients[clientName]` — relies on `createAndStore` to have already mutated the state by the time `reconnectClient` resolves. That ordering contract is implicit; a one-line comment at line 663 stating "createAndStore commits to InstanceState before resolving" would prevent future regressions.
- `packages/opencode/src/mcp/index.ts:660` — `if (!isTransportError(e)) throw e` rethrows synchronously inside an async catch. Correct, but the current code uses a `.then(...).catch(async (e) => { ... })` chain instead of `try/await`. An `async/await` rewrite of this 13-line block would be more legible without changing behavior.
- `packages/opencode/src/mcp/transport-error.ts:1-48` (new file, +48) — classifier accepts StreamableHTTPError (400/404/410/500, SDK code -1), Bun `ConnectionRefused`, Node `ECONNREFUSED`/`ECONNRESET`, Undici `UND_ERR_SOCKET`, and a fetch-failed text fallback. Reasonable surface; the `code === -1` path is fragile if upstream SDK ever assigns -1 to a non-transport error (e.g. cancellation). The accompanying `transport-error.test.ts` (+102) does cover this case explicitly per the PR body.
- `packages/opencode/test/mcp/transport-error-probe.mjs:1-244` (new) — real fork-and-restart probe wired into `.github/workflows/test.yml:88-118` as a new job on Ubuntu + Windows. Good defense against SDK error-shape drift. Two-minute timeout is tight for Windows cold starts; consider 4-5 minutes.
- `packages/opencode/package.json:186` — `}` is followed by `\ No newline at end of file` per the diff. Repo style elsewhere is trailing newline.
- `packages/opencode/test/mcp/xingtian-mcp-server.mjs` (+221) — manual E2E. The supervisor/worker script is genuinely useful for repro, but 221 lines of test-only code in `test/mcp/` is heavy. Consider moving it under `script/dev/` or `examples/`, since it's not invoked by CI.
- `packages/opencode/src/mcp/index.ts:216-217` — adds `layerBridge = yield* EffectBridge.make()` and the `reconnecting` Map at layer construction time. Layer-scoped state, so a layer rebuild (test isolation) gives a fresh map. Correct.

## Verdict

`merge-after-nits` — solves a real long-tail UX problem with a careful single-flight design and meaningful test coverage. Land after: (1) trailing newline on `package.json`, (2) one-line comment on the reconnect→state-read ordering contract, (3) tighten or relax the Windows probe timeout, (4) consider relocating `xingtian-mcp-server.mjs` out of `test/mcp/`.
