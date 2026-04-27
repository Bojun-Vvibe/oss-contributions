# QwenLM/qwen-code PR #3617 — fix(core): split tool-result media into follow-up user message for strict OpenAI compat

- Link: https://github.com/QwenLM/qwen-code/pull/3617
- Head SHA: `f64af202cac93b0eb220262817790c3d74a98410`
- Size: +134 / -0 across 4 files

## Summary

Adds an opt-in `splitToolMedia: boolean` flag to `ContentGeneratorConfig` (`contentGenerator.ts:116-128`) that, when true, scans the converted `role: "tool"` message in the OpenAI content-converter for `image_url` / `input_audio` / `video_url` / `file` parts, strips them out (replacing the tool message content with the joined text or a placeholder), and emits a separate `role: "user"` message immediately after carrying just `[ {type: 'text', text: '(attached media from previous tool call)'}, …mediaParts ]`. Closes #3616 (LM Studio rejecting multimodal tool-result content with HTTP 400 "Invalid 'messages' in payload").

## Specific-line citations

- `core/openaiContentGenerator/converter.ts:444-491`: the gating `if (requestContext.splitToolMedia && Array.isArray(toolMessage.content))` correctly defaults off and only triggers the split when content is array-shaped (string-shaped tool messages already pass strict validation). The text/media partition is a single-pass loop over the array, which is the right shape — no double iteration or mutation-during-iteration.
- `core/openaiContentGenerator/converter.ts:485-490`: when text parts exist, `toolMessage.content = textOnly` collapses array-shaped content back to a string, which is exactly what the OpenAI spec wants for tool messages. When only media existed, the placeholder `'[media attached in following user message]'` keeps the tool message non-empty (LM Studio also rejects empty-string tool content). Both branches are correct.
- `core/openaiContentGenerator/converter.ts:493-503`: the follow-up user message uses `as unknown as OpenAI.Chat.ChatCompletionContentPartText[]` cast — this is honest about the OpenAI SDK's type layout (`ContentPartText[]` is what the field is typed as for `role: "user"`, but it's actually `ContentPart[]` at runtime). Worth a comment but not blocking.
- `pipeline.ts:524`: `splitToolMedia: this.contentGeneratorConfig.splitToolMedia ?? false` propagates the flag into `RequestContext` with explicit-false default, so absence in config = current behavior. Backward compat preserved.
- `converter.test.ts:385-457`: the new test mirrors the existing embedded-image test exactly, just with `strictContext: { …requestContext, splitToolMedia: true }`, then asserts (a) `typeof toolMessage.content === 'string'` and `.toContain('Image content')` and (b) the follow-up user message contains the `image_url` part with the correctly-formed `data:image/png;base64,…` URL. Pins both halves of the contract; doesn't pin the "tool message had only media → placeholder" branch but that's a small gap.

## Verdict

**merge-as-is**

## Rationale

The design is right: the spec issue is real (OpenAI tool messages are string / text-part only), permissive servers happen to accept the violation, and a strict opt-in flag preserves both audiences without forcing a breaking default. The implementation is contained to the converter (no provider-detection logic creeping into the pipeline), the follow-up-user-message shape matches what OpenAI expects for inline media generally, and the placeholder text on the tool message side is a thoughtful touch that keeps the tool→model causal chain readable in transcripts and logs.

The "only-media" branch coverage gap (text array → placeholder string) is worth a follow-up test but is not load-bearing for the bug being fixed (the LM Studio reproducer in #3616 has the standard text-plus-image shape). The `as unknown as ContentPartText[]` cast at `:497` deserves a one-line comment ("OpenAI SDK types ContentPart[] as ContentPartText[] for user role; runtime accepts image_url here") so a future reader doesn't try to "fix" the type. Both are nits, not merge conditions.

