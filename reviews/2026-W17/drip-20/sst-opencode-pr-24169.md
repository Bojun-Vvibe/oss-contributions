---
pr_number: 24169
repo: sst/opencode
head_sha: 74dac92bf5426b1b9b2afe2254e30e3dc4bf5a35
verdict: merge-as-is
date: 2026-04-25
---

# sst/opencode#24169 — `refactor(schema): decode effect schemas directly`

**What changed.** +24 / −19 across four files. Mechanical replacement of `<EffectSchema>.zod.<safeParse|parse>` calls with cached `Schema.decodeUnknownSync` / `Schema.decodeUnknownResult` decoders, removing the `effect-zod` adapter dependency at the call sites:

- `packages/opencode/src/acp/agent.ts:49` — defines `decodeTodos = Schema.decodeUnknownResult(Schema.fromJsonString(Schema.Array(Todo.Info)))` once at module scope; lines 376 and 905 swap `z.array(Todo.Info.zod).safeParse(JSON.parse(part.state.output))` for `decodeTodos(part.state.output)`. Result branching becomes `Result.isSuccess(parsedTodos) ? parsedTodos.success : parsedTodos.failure` (was `.data` / `.error`).
- `packages/opencode/src/cli/cmd/import.ts:16-17` — adds module-scope `decodeMessageInfo = Schema.decodeUnknownSync(MessageV2.Info)` and `decodePart = Schema.decodeUnknownSync(MessageV2.Part)`. Lines 175 and 191 replace `.zod.parse(...)` with the cached decoder + `as MessageV2.Info | MessageV2.Part` assertions.
- `packages/opencode/src/control-plane/adaptors/worktree.ts:11` — drops `withStatics(s => ({ zod: zod(s) }))` from the `WorktreeConfig` schema and adds `decodeWorktreeConfig = Schema.decodeUnknownSync(WorktreeConfig)`. Three call sites (lines 26, 38, 42) now call `decodeWorktreeConfig(info)` instead of `WorktreeConfig.zod.parse(info)`. Imports of `zod` from `@/util/effect-zod` and `withStatics` from `@/util/schema` are removed.
- `packages/opencode/src/server/routes/instance/pty.ts:13` — adds `decodePtyID = Schema.decodeUnknownSync(PtyID)`. Line 176 swaps `PtyID.zod.parse(c.req.param("ptyID"))` for `decodePtyID(c.req.param("ptyID"))`. Import of `Effect` is widened to `Effect, Schema`; `zod` import is dropped.

**Why it matters.** `.zod` was a compatibility shim that runs every parse through Zod-via-effect-zod, which (a) doubles the runtime work for each decode, (b) loses Effect's structured `ParseError`s in favor of Zod's flatter `ZodError`, (c) holds a reference to the Zod runtime even on call sites that never need it. Hoisting the decoder to module scope (rather than re-deriving in each invocation) avoids per-call allocator pressure too.

**Concerns.**
1. **`as MessageV2.Info` / `as MessageV2.Part` casts (`import.ts:175, 191`)** suggest `Schema.decodeUnknownSync` returns a wider type than needed. If the schema is the `MessageV2.Info` schema, the decoder should infer the exact type. The cast is probably needed because `MessageV2.Info` is a union or has variant tags that `decodeUnknownSync`'s inference broadens. Worth a comment, but not blocking — the runtime check still happens.
2. **`Schema.fromJsonString(Schema.Array(Todo.Info))`** (acp/agent.ts:49) — combined JSON-string parse + decode into a `Result`. Previously `JSON.parse` happened *outside* the safeParse, meaning a non-JSON output would throw and the catch-block log never ran. Now the decoder catches the JSON parse failure too and routes it through the `Result.isFailure` branch. Net positive: malformed `todowrite` outputs now log "failed to parse todo output" instead of crashing the message-decoder loop.
3. **`parsedTodos.failure`** is logged directly (`{ error: parsedTodos.failure }`). For Effect's `ParseError`, that's a structured tree; consumers of these logs (presumably JSON-line) will get a multi-key object instead of the prior `ZodError.message` string. If any log-aggregator alerting is keyed on the error string shape, it'll need updating. Cheap to verify.
4. **Pty WebSocket boundary (`pty.ts:176`)** — if `decodePtyID(c.req.param("ptyID"))` throws on bad input, the request returns a 500 with the raw exception. Previously `.zod.parse` threw a `ZodError` which most server middleware caught into a 400. Confirm there's a top-level error handler on the upgrade websocket route that maps `ParseError` to a 400, otherwise this is a small status-code regression for clients sending malformed PTY IDs.
5. **Test coverage in body** lists `bun typecheck`, three `bun run test:ci` files (`import`, `worktree`, `worktree-remove`, `pty-session`), and a push hook `bun turbo typecheck`. No test added for the `acp/agent.ts` todo-decode change — the test suite doesn't cover ACP `todowrite` round-trips per the listed files. Acceptable for a like-for-like refactor; flag if a follow-up adds coverage.

Tight, well-scoped refactor — moves decode work out of the hot path and removes one layer of adapter complexity. Land it.
