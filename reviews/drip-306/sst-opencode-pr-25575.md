# Review: sst/opencode #25575 — fix: propagate chunkTimeout to streamText timeout.chunkMs

- **Repo**: sst/opencode
- **PR**: #25575
- **Head SHA**: `cd419177f81f9963a812d5158b0a9bcc4df7fdf5`
- **Author**: Fatty911

## What it does

Plumbs the existing `chunkTimeout` provider option down into the AI SDK's
`streamText({ timeout: { chunkMs: ... } })` call so that silent SSE
dropouts (TCP open, no chunks) actually trigger an abort instead of
hanging forever. Also documents the option (EN + zh-CN).

## Diff notes

`packages/opencode/src/session/llm.ts:333-340` — extracts
`chunkTimeout` from the per-call `options` bag, gated on `typeof === "number"`,
debug-logs when configured. Then at `packages/opencode/src/session/llm.ts:377-382`,
it spreads `{ timeout: { chunkMs: chunkTimeout } }` into the `streamText`
call only when truthy. This is correct: the `?:` guard avoids passing
`{ chunkMs: 0 }` (which the AI SDK could interpret as "fire immediately")
and avoids passing `{ chunkMs: undefined }` (harmless, but noisy).

Doc changes at `packages/web/src/content/docs/config.mdx:381` and
`zh-cn/config.mdx:241,259` — recommends 15–30s, calls out that this catches
TCP-alive-but-stream-dead failures. The Chinese doc was missing the
`chunkTimeout` line entirely; this adds it. Good parity work.

## Concerns

1. The new option is read with bracket access (`options["chunkTimeout"]`)
   rather than typed. That's consistent with how the surrounding code
   treats provider options (untyped passthrough), but if the schema for
   provider options is generated/checked elsewhere, a typed shape would
   be safer. Non-blocking.
2. The recommended floor (15s) is documented but not enforced. A user
   setting `chunkTimeout: 50` would get aborts on every realistic stream.
   A clamp / warning below ~1000ms would be a nice follow-up; not
   required for this PR.
3. No new test. The change is small enough and the AI SDK behavior is
   already covered upstream, but a unit test that asserts
   `streamText` is called with `timeout.chunkMs` when the option is set
   would prevent regression. Consider asking for one before merge.

## Verdict

merge-after-nits
