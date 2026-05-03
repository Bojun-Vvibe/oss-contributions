# sst/opencode #25549 — feat(provider): add Featherless AI provider

- URL: https://github.com/anomalyco/opencode/pull/25549
- Head SHA: `30339426e965e9238fcba93e6ff6f0086bbd1064`
- Author: ArEnSc
- Scope: +1632 / 0 across 10 files (all new)

## Summary

Adds Featherless.ai as a provider with a client-side concurrency gate. Featherless
bills by per-model concurrency cost; firing N parallel streams without local
gating produces a wall of 429s. The gate sits between the AI SDK and `fetch`,
queues locally based on a periodically-refreshed cost cache, and releases slots
when the response stream actually closes (~60ms after last chunk on the wire).

## What I checked

- `packages/opencode/src/provider/sdk/featherless/index.ts` — exports
  `createFeatherlessFetch({ apiKey })` returning a `fetch`-shaped function.
  Integration into `provider.ts` is exactly 23 lines, mirroring the existing
  `google-vertex` pattern of injecting `options.fetch`. Surface area is small
  and easy to revert.
- `gate.ts` (283 lines) — implements: snapshot-on-start, FIFO waiter queue,
  SSE listener that tails `/account/concurrency` to drive drains,
  reconnect-on-stream-close, idempotent release. The contract is captured
  in `gate.test.ts` (377 lines, 21+ cases). The release-on-stream-close
  detail (vs release-on-headers-received) is documented in the PR body and
  is the right call — billing on the upstream side is held for the full
  stream duration.
- `concurrency-cache.ts` — handles cold-start (no snapshot yet → fetch one),
  corrupt cache (recovers gracefully), and stale entries. 271 lines of tests
  back this up.
- `stream-release.ts` (57 lines, 107 lines of tests) — wraps the `ReadableStream`
  so that natural completion, consumer cancel, and read error all release
  exactly once. Critical for not leaking slots; the "exactly once" guarantee
  is asserted directly in tests.
- `seed.ts` — bundles 11 popular models as a synthetic config-provider entry
  so users see Featherless in `/connect` without editing `opencode.json`.
  PR body commits to removing this once a `models.dev` PR lands. Reasonable
  short-term tradeoff; please file the follow-up issue.
- `script/featherless-smoke.ts` — live smoke test against the real API
  (requires `FEATHERLESS_API_KEY`). Useful for manual validation but is not
  part of CI; that's appropriate given it hits a paid third-party endpoint.

## Concerns / nits

1. **Footprint vs typical provider PR.** Most providers in this repo are 1
   adapter file + a `models.dev` entry. This PR ships 1632 lines for a single
   provider because of the gate. The architecture is well-isolated under
   `sdk/featherless/`, but the maintenance cost is higher than other providers.
   Worth a maintainer thumbs-up that this complexity belongs in-tree vs a
   plugin.
2. **SSE listener resilience.** `gate.ts` reconnects on stream close; verify
   it backs off on repeated reconnect failures so a Featherless outage can't
   spin a tight reconnect loop. Quick read suggests there's a small
   reconnect delay but no exponential backoff — worth confirming.
3. **API key handling.** `createFeatherlessFetch({ apiKey })` captures the
   key in a closure and uses it for the SSE subscription too. Confirm the
   key is read fresh on rotation if opencode supports key rotation mid-session
   (looks like other providers re-resolve per-request via the SDK builder, so
   this might already be handled by `provider.ts` rebuilding the fetch).
4. **Seed maintenance.** The 11 hardcoded models in `seed.ts` will drift.
   Adding a `// TODO(remove-after-models.dev-pr)` comment with an issue
   reference would make the temporary nature explicit.
5. Smoke script is 171 lines of Bun; would benefit from a short README note
   in `packages/opencode/script/` explaining when to run it.

## Risk

Medium. Net-new code, isolated under `sdk/featherless/`, behind opt-in
config. No risk to existing providers. The concurrency gate is the hairy
part — well tested but novel concurrency code is the kind of thing that
shows its bugs in production.

## Verdict

`merge-after-nits` — primarily: confirm SSE reconnect backoff, add seed-
removal TODO with issue link, get maintainer alignment on the in-tree
gate footprint.
