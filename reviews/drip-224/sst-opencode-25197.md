# sst/opencode #25197 — fix(opencode): Bedrock non-Anthropic normalize and xhigh to max effort

- **Repo:** sst/opencode
- **PR:** https://github.com/sst/opencode/pull/25197
- **HEAD SHA:** `ddc7f5297e05824f685c900d0e15dee5f2efc15b`
- **Author:** jackmazac
- **Verdict:** `merge-after-nits`

## What the diff does

Two-file fix for `@ai-sdk/amazon-bedrock` provider correctness, closing #25193:

1. `packages/opencode/src/provider/transform.ts:217-258` — adds a new
   `normalizeMessages` arm guarded by `model.api.npm === "@ai-sdk/amazon-bedrock"
   && !model.api.id.includes("anthropic") && !model.api.id.includes("mistral")`
   that, for Bedrock models *other than* Anthropic/Mistral adapters:
   - Strips `cachePoint` provider-options from both the message-level
     `providerOptions` and each part's `providerOptions`, scanning two key
     namespaces (`"bedrock"` and the dynamic `model.providerID`).
   - Filters out `reasoning` and `redacted-reasoning` parts from assistant
     messages.
   - Inserts a `{ type: "text", text: "..." }` placeholder if the assistant
     message would otherwise be empty after the reasoning filter.

2. `packages/opencode/src/provider/transform.ts:725` — flips
   `maxReasoningEffort: effort` to `maxReasoningEffort: effort === "xhigh"
   ? "max" : effort` for adaptive reasoning variants (Bedrock's adaptive
   reasoning expects `max`, not `xhigh`, on Opus 4.7).

3. `packages/opencode/test/provider/transform.test.ts:3007-3022` — renames
   the test arm and pins the `"xhigh"` → `"max"` mapping plus the existing
   `display: "summarized"` for opus-4-7.

## Why the change is right

The fix lives at **the right layer**: `normalizeMessages` is the
provider-export transform boundary that runs once per request before
`generate()`, so a single guarded arm here covers every call site (chat,
sub-agent, compact). The two-key namespace scan (`["bedrock",
model.providerID]`) at `transform.ts:221` is the load-bearing detail —
`cachePoint` can be set under either the canonical `bedrock` namespace or
the user's provider ID, and missing one leaks the option to the
non-Anthropic Bedrock adapter, which rejects it.

The empty-message-placeholder at `:241-242` is the right defensive shape:
some Bedrock non-Anthropic adapters reject assistant messages with zero
content parts, and an assistant turn that was *only* reasoning becomes
empty after the filter — without the `"..."` placeholder this fix would
trade one Bedrock failure for another.

The `xhigh → max` mapping at `:725` is a pure value translation at the
provider-export boundary; the prior code passed the UI-layer `xhigh` token
straight through to Bedrock which has no such enum value.

## Nits (non-blocking)

1. **No test for the non-Anthropic Bedrock normalize path.** The test diff
   only updates the existing adaptive-reasoning test (which is the
   `xhigh→max` arm). The much larger `normalizeMessages` arm at
   `transform.ts:217-258` ships with zero direct coverage — the empty-message
   placeholder, the cachePoint-strip on both namespaces, and the
   reasoning-part filter are all untested. At minimum, one test asserting
   "assistant message with only `{type: 'reasoning'}` parts becomes
   `[{type: 'text', text: '...'}]`" would pin the load-bearing
   placeholder behavior so a future filter-only refactor can't re-introduce
   the empty-message regression.

2. **`!model.api.id.includes("anthropic")` is a substring guard.** A Bedrock
   adapter id like `bedrock/cohere-anthropic-bridge-…` (hypothetical but
   plausible) would silently take the *non-Anthropic* path despite hosting
   an Anthropic-shaped model. A `model.api.id.startsWith("anthropic/") ||
   model.api.id.includes("/anthropic-")` shape would be safer. Same for
   the mistral check.

3. **`Object.keys(rest).length > 0 ? rest : undefined` shape on `:226`** —
   the surrounding code spreads `out = {...out, [k]: ...undefined}` which
   leaves the key with an explicit `undefined` value rather than removing
   it. Some downstream serializers omit-undefined, others emit
   `"k":null` — worth a quick check that the `@ai-sdk/amazon-bedrock`
   serializer treats both equivalently, otherwise an explicit
   `delete out[k]` arm is safer than the spread.

4. **Test name still says "anthropic opus 4.7" at `:3007`** but the new
   assertion is the cross-cutting `xhigh → max` mapping that applies to
   *every* adaptive-reasoning model, not just Opus 4.7. A second test arm
   for a non-opus adaptive model would prevent a future Opus-only narrowing
   from silently regressing the cross-cutting mapping.

## Verdict rationale

Right-shaped two-pronged fix at the right boundary, with the load-bearing
empty-placeholder and dual-namespace-key details correctly handled. The
`xhigh → max` mapping is pinned by test, but the much larger
`normalizeMessages` arm needs at least one direct unit test before this is
`merge-as-is`.

`merge-after-nits`
