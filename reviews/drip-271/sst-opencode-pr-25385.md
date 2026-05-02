# sst/opencode #25385 — feat(provider): repair malformed SSE JSON via jsonrepair

- URL: https://github.com/sst/opencode/pull/25385
- Head SHA: `d2c17f59cd78ca9eb2cdadfd5d40b2b27842dad8`
- Author: @water-in-stone
- Stats: +~250 / -3 across ~5 files (new `sse-repair.ts` + tests, config
  flag, provider wiring)

## Summary

Adds an opt-in `experimental.enable_sse_json_repair` flag. When set, a
`TransformStream` sits between the upstream SSE response and the AI
SDK's `parseJsonEventStream`, splitting on `\n\n`, fast-pathing valid
chunks, and only invoking `jsonrepair` when `JSON.parse` fails. Verified
parses are committed; unrepairable chunks pass through unchanged so the
downstream parser still surfaces the original error.

## Specific feedback

- `packages/opencode/src/provider/sse-repair.ts:21-58` — the fast-path
  ordering (`JSON.parse` first, `jsonrepair` only on failure) is the
  right call: jsonrepair on every chunk would dominate the streaming
  hot path. Good.
- `packages/opencode/src/provider/sse-repair.ts:30-32` — round-tripping
  `JSON.parse(jsonrepair(payload))` before substitution is a smart guard
  against jsonrepair's known habit of producing "valid-but-weird" JSON.
  Worth a code comment pointing at a known case so future maintainers
  don't simplify it away.
- `packages/opencode/src/provider/sse-repair.ts:60-95` — `TransformStream`
  buffer/flush handles cross-chunk events correctly and the flush path
  drains a trailing fragment without `\n\n` (defensive against
  non-spec-conforming servers). Nice.
- `packages/opencode/src/provider/sse-repair.ts:74` — the new `Response`
  inherits `headers/status/statusText` but the body is the transformed
  stream. `Headers` clone is correct; consider also forwarding `url` so
  downstream debug logs aren't blank (AI SDK reads `res.url` in some
  paths).
- `packages/opencode/src/provider/provider.ts:1591-1597` — `cfg` is read
  per `getSDK` call. That's fine — `getSDK` is cached on `s.models` —
  but worth a comment noting this so a refactor doesn't accidentally
  hoist it into the per-request path.
- Default-off behind `experimental.*` is exactly the right rollout
  posture for a stream-mutating shim. Good.
- `packages/opencode/test/provider/repair-sse.test.ts:1-107` — covers
  valid passthrough, `[DONE]`, non-data lines, and the actual Z.AI
  payload. Missing: a multi-chunk test where the malformed event is
  split across two `transform` calls (the buffer/flush branch is the
  trickiest part of this PR and currently uncovered).

## Verdict

`merge-after-nits` — well-engineered, narrowly scoped, opt-in. Add the
chunked-boundary test and a brief code comment about the round-trip
guard, and this is shippable.
