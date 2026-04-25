# sst/opencode #24262 — fix(provider): inject chat_template_kwargs for Nvidia NIM deepseek-v4 models

- **Repo**: sst/opencode
- **PR**: [#24262](https://github.com/sst/opencode/pull/24262)
- **Head SHA**: `15274deef98a25e8593d16d4c0f881a1a1fea274`
- **Author**: Zireael
- **State**: OPEN (+13 / -1)
- **Verdict**: `request-changes`

## Context

Author wants Nvidia NIM-hosted DeepSeek-v4 to receive
`chat_template_kwargs: { enable_thinking: true, thinking: true }`
so reasoning content comes through. The fix is wedged into the
OpenRouter `custom` provider loader as a custom `fetch` wrapper
that string-matches request bodies.

## Design

Single change at `packages/opencode/src/provider/provider.ts`:

1. **Line 419**: Header rename
   `"X-Title": "opencode"` → `"X-title": "opencode"`. Unrelated
   to the stated fix and arguably wrong (HTTP header names are
   case-insensitive on the wire but OpenRouter docs use
   `X-Title`; this churns history for no benefit and may break
   anyone matching headers case-sensitively in tests).

2. **Lines 421-431**: New `fetch` override on the custom
   provider that:
   - Checks `opts.body` is a string and contains the literal
     substring `"deepseek-v4"`.
   - `JSON.parse`s the body, checks `body.model` includes
     `"deepseek-v4"`, mutates in-place to add
     `chat_template_kwargs`, and re-serializes.
   - Falls back to `globalThis.fetch(url, opts)`.

## Risks

- **Wrong code path.** This change is inside the `custom`
  loader's OpenRouter-shaped headers block (the `HTTP-Referer`
  / `X-Title` are OpenRouter-specific). Nvidia NIM is not
  OpenRouter — so either the user is routing NIM traffic
  through OpenRouter (in which case OpenRouter, not opencode,
  should handle the param), or they've hand-configured a
  `custom` provider pointing at NIM and inherited the
  OpenRouter-shaped headers, which is a config bug to fix in
  config, not in the provider loader.
- **String-matching `"deepseek-v4"` in request bodies is
  unsafe.** Any chat content that mentions "deepseek-v4"
  (a user asking "how does deepseek-v4 work?") triggers the
  JSON parse and mutation. Should be a model-id prefix check
  on the parsed body, not a substring scan on the serialized
  body.
- **Silent `try/catch(e) {}`** at line 428 swallows
  `JSON.parse` errors. Any non-JSON body (multipart, streamed)
  silently bypasses the wrapper. At minimum log; better,
  inspect `Content-Type` instead of trying to parse blindly.
- **Mutating the parsed body and re-serializing changes
  property order, drops `undefined` fields, and may strip a
  trailing newline.** For providers that sign request bodies
  (NIM with HMAC, e.g.), this would invalidate the signature.
  Don't re-serialize if you don't have to.
- **Hard-codes the `chat_template_kwargs` payload** with both
  `enable_thinking: true` and `thinking: true`. The redundant
  pair suggests the author isn't sure which key NIM honors —
  one is right and the other is dead weight or, worse,
  rejected. Pick one based on Nvidia NIM docs.
- **`X-Title` typo (line 419)** breaks attribution in
  OpenRouter's dashboard. OpenRouter is case-insensitive on
  receive but uses `X-Title` consistently in docs; some
  middleware does respect case. Revert this.
- **Per-request `JSON.parse` + `JSON.stringify` on every chat
  call** is a non-trivial cost on a hot path. Even the cheap
  case (no `"deepseek-v4"` substring) pays for a `String.includes`
  scan over the entire body. Move the wrapper behind a config
  flag or model-id check at provider construction time.
- **No test added.** A regression here is invisible until a
  user reports "thinking output disappeared" months later.

## Suggestions

The right shape is roughly:
- Add `extraBody.chat_template_kwargs` (or whatever the AI SDK
  calls it) as a per-model config option in the provider's
  model-options layer, not as a body-mutating fetch wrapper.
- If the wrapper *must* live here, gate it on
  `body.model?.startsWith("deepseek-v4")` after a guarded
  `JSON.parse`, behind a `Content-Type: application/json`
  check, and log the parse failures.
- Drop the `X-Title` rename (revert to `"X-Title": "opencode"`).
- Pick one of `enable_thinking` / `thinking` based on NIM docs.
- Add a unit test against a mocked `fetch` proving the wrapper
  injects when `model` matches and is a no-op when it doesn't.

## What I learned

"Inject extra params for one specific model on one specific
provider" is one of the most frequent feature requests in
multi-provider proxies, and it almost always wants to live in a
per-model config slot, not in a body-mutating fetch wrapper.
The wrapper approach scales O(N) configs into O(N) fetch
overrides, each with its own substring scan, and they compose
poorly. A `extraRequestBody: Record<string, unknown>` on the
model definition is the durable shape.
