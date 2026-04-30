# sst/opencode #25101 — fix(provider): show opus 4.7 thinking chunks

- Head SHA: `63c04839b3b0e7724db043a456740ee060401fa9`
- Files: `packages/opencode/src/provider/transform.ts`, `packages/opencode/test/provider/transform.test.ts`
- Size: +105 / -8

## What it does

Opus 4.7 omits visible thinking text unless the request explicitly opts in via
`thinking.display: "summarized"`. Previously this opt-in was scattered across
three call sites in `variants(...)` using string-literal `apiId.includes(...)`
checks (transform.ts:653, :687), and the default `options(...)` builder did not
set it at all — so the fresh-session, no-variant path silently dropped Opus 4.7
reasoning chunks.

The fix introduces `isAnthropicOpus47(apiId)` (transform.ts:430-433) that
lowercases once and checks both `opus-4-7` and `opus-4.7` spellings, then
swaps it in at all three pre-existing sites (transform.ts:434, :653, :687) and
adds a fourth at the new variants xhigh/max effort branch (transform.ts:493).
The genuinely new behavior is in `options(...)`: lines 916-931 add a default
`thinking: { type: "adaptive", display: "summarized" }` for the four
`@ai-sdk/anthropic`, `@ai-sdk/google-vertex/anthropic`, `@ai-sdk/gateway` SDKs
and a parallel `reasoningConfig` for `@ai-sdk/amazon-bedrock`.

## Coverage

The new `describe("ProviderTransform.options - opus 4.7 thinking", ...)` block
(transform.test.ts:380-432) covers anthropic and bedrock paths with explicit
assertions on the result shape. The gateway test at transform.test.ts:359-371
locks the gateway-specific `caching: "auto"` co-existence. The variants test
update at transform.test.ts:2532-2545 confirms the previously-existing
`xhigh`/`max` variants now also carry `display: "summarized"` (the previous
inconsistency where xhigh/max omitted display while low/medium/high included
it).

## Concerns

1. **Vertex test missing.** `@ai-sdk/google-vertex/anthropic` is in the
   `.includes` list at transform.ts:919 but only anthropic and bedrock have
   focused option-builder coverage. A two-line vertex test mirroring the
   anthropic one would lock the third SDK explicitly.
2. **Substring match.** `isAnthropicOpus47("claude-opus-4-70")` would return
   true if Anthropic ever ships a `4.70` snapshot id. Cheap to harden with
   a trailing-boundary regex, though current Anthropic naming makes a `.70`
   collision unlikely.
3. **Implicit override risk.** The default block at transform.ts:916-928
   unconditionally writes `result["thinking"]`/`result["reasoningConfig"]` —
   if any earlier caller in `options(...)` had already populated those keys
   with a *non*-summarized config, this silently clobbers it. A `?? result.thinking`
   merge or guard would be safer.

The change closes a real product bug (no thinking visible by default on the
flagship reasoning model) and the test surface is proportional. Nits are
non-blocking polish.

Verdict: merge-after-nits
