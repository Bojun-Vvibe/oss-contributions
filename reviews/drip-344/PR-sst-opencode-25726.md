# sst/opencode#25726 — fix(server): provide fresh ConfigProvider per HttpApi listener

- PR ref: `sst/opencode#25726`
- Head SHA: `ea155b4c1423a9f68bdc9fe43c84dc92ee9a74a2`
- Title: fix(server): provide fresh ConfigProvider per HttpApi listener
- Verdict: **merge-as-is**

## Review

The root cause analysis here is sharp and the fix is appropriately scoped. Effect's
default `ConfigProvider` snapshots `process.env` on first read and caches it on a
module-level `Reference`, so on a second `Server.listen()` call any
`Config.string(...)` derived value would still observe the *original* environment.
Installing a fresh `ConfigProvider.fromEnv()` layer per listener at
`packages/opencode/src/server/server.ts:262` is the smallest possible fix that
restores per-listener environment evaluation without touching call sites.

The comment above the `Layer.provide` (`packages/opencode/src/server/server.ts:264-268`)
is excellent — exactly the kind of "why, not what" inline note future maintainers will
need when they wonder whether this layer is redundant with Effect defaults.

The test changes in `packages/opencode/test/server/httpapi-listen.test.ts:43` make
`startNoAuthListener` parametric and then loop the no-auth ticket regression across
both `effect-httpapi` and `hono` backends (`test:303-318`). That's a proper
generalization — the previous version only exercised the hono path because the helper
unconditionally set `OPENCODE_EXPERIMENTAL_HTTPAPI = false`. No nits.
