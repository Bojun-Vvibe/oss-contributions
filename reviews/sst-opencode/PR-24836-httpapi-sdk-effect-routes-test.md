# sst/opencode#24836 — test(httpapi): cover sdk effect routes

- **Repo:** [sst/opencode](https://github.com/sst/opencode)
- **PR:** [#24836](https://github.com/sst/opencode/pull/24836)
- **Head SHA:** `ae07f6470e7b032fca0685b32346c62460f168e9`
- **Size:** +79 / -0 (one new file)
- **State:** OPEN
- **File touched:** `packages/opencode/test/server/httpapi-sdk.test.ts`

## Context

This PR is the test-coverage cap for the multi-PR Effect HttpApi rollout that
opencode has been running over the last week (drip-148's #24827 normalized SDK
schemas; #24835 — reviewed in parallel with this PR — wires the global/control
groups). It exercises the freshly-wired routes by going through the **generated
SDK** (`@opencode-ai/sdk/v2`) rather than hitting the handler with a hand-built
`Request`. That's the right level — it pins both the Effect runtime *and* the
SDK signature surface in one test pass.

## Design analysis

The clever bit is the in-process `fetch` shim:

```ts
// httpapi-sdk.test.ts:13-18
const handler = ExperimentalHttpApiServer.webHandler().handler
const fetch = ((request: RequestInfo | URL, init?: RequestInit) =>
  handler(new Request(request, init), ExperimentalHttpApiServer.context)) as typeof globalThis.fetch
return createOpencodeClient({ baseUrl: "http://localhost", directory: input?.directory, fetch })
```

By passing the Effect web handler in as the SDK's `fetch` implementation, the
test bypasses the network entirely — no port allocation, no TLS, no flake — but
still goes through every layer the SDK applies (path templating, query
serialization, response parsing, SSE iteration). That's exactly the parity
guarantee `httpapi-sdk-test` should be giving.

The `expectStatus` helper at line 26 is the right shape for the negative
assertions (`auth.set({ providerID: "test" })` expecting 400, `global.upgrade`
with a wrong-typed target expecting 400) — it forces the test to await *both*
the SDK promise and read `.response.status`, which catches the case where the
SDK swallows the body but exposes a useful response object.

The SSE smoke at lines 51-54 is well-judged:

```ts
const events = await sdk.global.event({ signal: AbortSignal.timeout(1_000) })
const first = await events.stream.next()
await events.stream.return(undefined)
expect(first.value).toMatchObject({ payload: { type: "server.connected" } })
```

It only asserts the first event (which #24835's `globalHandlers` emits
synchronously inside `start`) and tears the stream down promptly with
`stream.return(undefined)`. That pattern avoids the "test hangs because the
heartbeat keeps firing" trap.

## Risks / nits

1. `Flag.OPENCODE_EXPERIMENTAL_HTTPAPI = true` is set inside `client()` but only
   captured/restored once at module-load time (`const original =
   Flag.OPENCODE_EXPERIMENTAL_HTTPAPI` at line 11). If two tests in the same
   file ran concurrently against `client()` and one happened to read the flag
   between the assignment and a peer's `afterEach`, the captured `original`
   would already be `true` and the cleanup would be a no-op. Bun's `test`
   runner runs `describe` blocks serially by default so this is fine in
   practice — worth a one-line comment though.
2. The negative test for `global.upgrade({ target: 1 as unknown as string })`
   relies on the runtime decoder rejecting a numeric where the schema demands a
   string. That's load-bearing on `Schema.String` in `GlobalUpgradeInput`. If
   anyone ever loosens that to `Schema.Union(Schema.String, Schema.Number)`,
   this test will silently start asserting a 200. Adding an extra assertion
   (`expect(body).toMatchObject({ success: false, error: ... })`) would lock
   the contract more tightly.
3. The `instance routes` test at line 56 covers `find.files`, `config.get`,
   `config.providers`, `project.current`, `session.create`, `session.list`,
   `file.read` — but not the more failure-prone routes (`session.delete`,
   `provider.update`). That's explicitly deferred ("Prompt/generation SDK
   coverage should use the repo mock LLM fixture in a separate follow-up"),
   which is the right call for this PR but worth tracking in a follow-up issue.

## Verdict

**merge-as-is.** This is exactly the right shape of test for the HttpApi
rollout: it goes through the SDK (catches schema/signature drift), uses the
in-process fetch shim (no port flake), and stays focused on routing /
validation rather than dragging in the LLM mock. The nits above are
follow-ups, not blockers.

## What I learned

- Passing `fetch` in to a generated SDK is the cleanest way to write
  in-process integration tests without standing up a server. Catches both
  serialization and routing bugs in one assertion.
- For SSE tests, calling `stream.return(undefined)` after the first event is
  the right way to tear down — `controller.cancel()` from the SDK side
  triggers the handler's `cancel` callback which clears the heartbeat.
- A captured `const original = ...` at module top is a fragile cleanup
  pattern when the flag is mutated inside a setup helper rather than in a
  `beforeEach`. Worth a comment when it's intentional.
