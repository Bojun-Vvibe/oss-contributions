# Review: anomalyco/opencode #25973 — feat(acp): extract system prompt from synthetic parts in session/prompt

- Carrier: `anomalyco/opencode`
- PR: #25973
- Head SHA: `22e486a6`
- Drip: 387

## Verdict
`merge-after-nits` (man)

## Summary
Surgical 12-line change at `packages/opencode/src/acp/agent.ts:1441-1490` that
splits ACP-incoming `parts` into a `system` string (collected from text parts
flagged `synthetic: true`) and `regularParts`, then forwards both fields
into the downstream `prompt(...)` call. The previous behavior concatenated
synthetic developer-supplied "system" content into the user-text body, where
it was indistinguishable from user prompt and bypassed the model provider's
dedicated system-message channel.

## Diff anchors
- `packages/opencode/src/acp/agent.ts:1444-1452` — new `regularParts` filter
  + `system` accumulator. Filter predicate `p.type === "text" && p.synthetic`
  is correctly defensive: non-text synthetic parts (image/file/etc) flow
  through to `regularParts` instead of being silently dropped.
- `packages/opencode/src/acp/agent.ts:1454` — `cmd` extractor switched from
  `parts` to `regularParts` so synthetic system text no longer pollutes the
  slash-command parser.
- `packages/opencode/src/acp/agent.ts:1488-1490` — both `parts: regularParts`
  and `system` are passed to the downstream call site so the provider adapter
  can route the system payload to its native channel (e.g., Anthropic's
  top-level `system`, OpenAI's `messages[0].role="system"`).

## Concerns / nits
1. The accumulator concatenates with `(system || "") + p.text + "\n"` which
   leaves a trailing `\n` even when only one synthetic part is present. Most
   provider transformers tolerate this, but the test suite should pin the
   exact serialized shape so a future reviewer knows it's intentional.
2. `system: undefined` vs `system: ""` semantics — if zero synthetic parts
   exist, `system` stays `undefined`, which is correct. But the filter inside
   the `cmd` extractor still walks `regularParts` instead of short-circuiting
   when `system === undefined && parts.length === regularParts.length`. Pure
   micro-perf nit.
3. No new test in this diff exercising the routing. A bun:test fixture
   feeding `[{type:"text", text:"hi"}, {type:"text", synthetic:true,
   text:"You are concise."}]` and asserting the downstream `prompt({system,
   parts})` call shape would lock the contract.
4. The `synthetic` flag's wire-format provenance isn't documented at the call
   site — a one-line `// synthetic parts are injected by ACP clients (e.g.
   editor-installed personas) and must bypass the slash parser` would help
   the next reader.

## Risk
Low. Pure split-and-route refactor with no semantic loss for non-synthetic
parts. The only consumer-visible behavior change is that synthetic text now
lands in the system channel instead of being prepended to the user message.
Provider adapters that don't yet honor the `system` field (if any) would
silently lose that text — worth a maintainer audit but the
`opencode-compatible` SDK already handles `system: string | undefined`.
