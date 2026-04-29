# sst/opencode PR #24887 — feat(tool): add hash tool for file checksums

- Repo: `sst/opencode`
- PR: https://github.com/sst/opencode/pull/24887
- Head SHA: `1e7acb637ff9d314b7873599d2dbb6994b71aa78`
- State: OPEN, +235/-0 across 6 files (closes #24886)

## What it does

Adds a built-in `hash` tool that streams a file off disk through `node:crypto.createHash` and returns a hex digest, with optional `expected` verification. Supports `sha256` (default), `sha512`, `sha1`, `md5`. Renders inline in the TUI via a new `Hash` `<Match>` arm next to the existing `Skill` arm.

## Specific reads

- `packages/opencode/src/tool/hash.ts:1-118` (new file) — Effect/Schema-shaped tool. The `Parameters` schema's `algorithm` uses `Schema.withDecodingDefault(Effect.succeed("sha256" as const))` (line 17-19) — correct; default is applied at decode time so a partial input from the model doesn't leak `undefined` into `createHash`.
- `tool/hash.ts:31-39` — `streamHash(filePath, algorithm, signal)` wires `AbortSignal` straight into `createReadStream`'s `{ signal: signal as any }` option. The `as any` is unnecessary on Node ≥ 18 (`createReadStream` accepts `signal` since v15.4) and worth dropping with a typed `ReadStreamOptions` import. More importantly, the promise's `error` handler accepts `reject` directly — fine, but on `'error'` the hasher is left intact in process memory; not a leak (gc collects it) but a comment that "abort propagates as `'error'` with `code === 'ABORT_ERR'`" would help the reader trust the cancellation contract.
- `tool/hash.ts:21-25` — `expected` doc says "non-hex characters (colons, spaces) are stripped" via `normalizeHex` (line 29: `s.toLowerCase().replace(/[^a-f0-9]/g, "")`). Correct shape, but `normalizeHex` doesn't *length-check* against the algorithm's expected digest length, so a user pasting a truncated `sha256` (say 32 hex chars instead of 64) gets a silent `matches: false` instead of a clear "expected length 64 for sha256, got 32" diagnostic. Worth a one-line guard.
- `packages/opencode/src/tool/index.ts` (registration) and the SDK/web TUI plumbing (4 of the 6 changed files) follow the existing `Skill` tool's pattern as a template — consistent with #24871's truncation handling and Web `<Match>` arm style. The TUI rendering at the unified `ToolPart` (~line 1593-1596): `<Match when={props.part.tool === "hash"}><Hash {...toolprops} /></Match>` slots into the existing tool-tag dispatch with no special-casing.
- TUI render component: the `state` memo (`metadata.matches === true ? "verified" : matches === false ? "mismatch" : "digest"`) collapses to "digest" on `undefined`. That's the right tristate, but the rendered string `"{algorithm} {state} {basename}"` produces `"sha256 mismatch foo.tgz"` with no visual emphasis on the *bad* state — a checksum-mismatch line should probably look distinct from "digest" (color cue, prefix marker) so users don't miss a verification failure during a streaming output flood.
- No test file in the diff. For a 118-line tool with five branches (default algo, explicit algo, expected match, expected mismatch, abort during stream), a unit test surface against tmp files would pin the metadata shape against future refactors.

## Risk

- No path-traversal/ external-dir gate visible in the diff snippet, but the tool imports `assertExternalDirectoryEffect` from `./external-directory` (line 7), which suggests the gate is wired — confirm it's invoked in the `Effect.gen` body before the `streamHash` call.
- Streaming with default 64KB highWaterMark on multi-GB files is fine for memory but ties up an event-loop slot for the duration. No timeout cap on the stream — a network-mounted file could block the tool indefinitely. Existing tools likely share this limitation; not a blocker.

## Verdict

`merge-after-nits` — clean addition that follows the established Effect-tool pattern and ToolPart rendering conventions. Three nits: (1) length-check `expected` against algorithm digest size for a clearer mismatch diagnostic, (2) drop the `as any` on `createReadStream` signal once the type-import is fixed, (3) add a minimal unit test surface for digest/match/mismatch since the metadata tristate is the consumer-facing contract. Optional: visual differentiation for `"mismatch"` state in the TUI render.
