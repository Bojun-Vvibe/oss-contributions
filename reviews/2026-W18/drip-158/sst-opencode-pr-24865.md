# sst/opencode#24865 — feat: add cors option to sdk ServerOptions

- PR: https://github.com/sst/opencode/pull/24865
- Head SHA: `17d34a64fa42cbeae66d403daecb7bd94762144a`
- Author: rodrigodmpa
- Diff: +10/-0 across `packages/sdk/js/src/server.ts` and `packages/sdk/js/src/v2/server.ts` (identical changes)

## What changed

Adds an optional `cors?: string | string[]` field to `ServerOptions` and pipes it through to the spawned `opencode serve` subprocess as one or more `--cors=<origin>` CLI args. The wiring at `server.ts:34-38` (and the mirror at `v2/server.ts:34-38`) is:

```
if (options.cors) {
  const origins = Array.isArray(options.cors) ? options.cors : [options.cors]
  for (const origin of origins) args.push(`--cors=${origin}`)
}
```

Placed between the existing `--log-level` push and the `launch(...)` call. No backend change — the receiving `--cors` flag must already exist on `opencode serve` for this to take effect (this PR is purely SDK plumbing).

## Observations

The one-string-or-array shape is the right ergonomics for this kind of option (cf. `cors` in fastify, hapi, koa-cors, etc.) and the conditional `if (options.cors)` correctly treats both `undefined` and the empty-string special case as "don't add" — though `cors: ""` would currently be falsy and silently dropped, which is the desired behavior since an empty origin would be nonsensical to pass to the server. `cors: []` is also handled correctly: `Array.isArray([])` is true, the `for` loop runs zero times, no flag is added.

The two files diverge only in their import path (`./process.js` vs `../process.js`); the change is byte-identical otherwise. That's the established pattern in this SDK for `v2/` shadow files, so the duplication is intentional. No new test surface added — accepted because the change is pure-plumbing string concat into an `args` array; the integration contract (does the server actually honor `--cors=<origin>`?) is owned by the server-side code and presumably already tested there.

Two missable nits: (1) no input validation/escaping. `cors: "evil\nX-Injected: yes"` would be naively concatenated into the spawn args. `child_process.spawn` with array-form args (which `launch(...)` presumably does) doesn't go through a shell so this is not a shell-injection vector, but the value still lands as the literal value of `--cors`, and depending on how the server parses it, a newline could be propagated into the eventual `Access-Control-Allow-Origin` header. A lightweight assertion (`/^https?:\/\/[^\s,]+$/` or equivalent) at the SDK boundary would catch ham-handed misuse. (2) The TSDoc on `ServerOptions` doesn't get a doc comment for the new field; a one-liner `/** Origin(s) the server should allow via Access-Control-Allow-Origin. */` would help editor tooltips and the generated `.d.ts`.

## Verdict

`merge-after-nits`
