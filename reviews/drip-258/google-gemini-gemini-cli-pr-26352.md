# google-gemini/gemini-cli PR #26352 — fix(core): filter unsupported multimodal types from tool responses

- PR: https://github.com/google-gemini/gemini-cli/pull/26352
- Head SHA: `77c7d7a7fedd4854ad15e071aeae965f622ae496`
- Author: @aishaneeshah (Aishanee Shah)
- Fixes: #25214

## Summary

The Gemini API returns `400 Bad Request` when a `functionResponse` part contains binary `audio/*` or `video/*` `inlineData` (images and PDFs are accepted). When a tool like `read_file` returned audio/video, the CLI passed the bytes through verbatim, which in YOLO/autonomous mode produced an infinite retry loop. This PR filters those parts in `convertToFunctionResponse` and injects a structured `[SYSTEM ERROR: PROTOCOL_LIMITATION]` steering message instructing the agent to use the `@path/to/file` syntax, which surfaces the file as a standard multimodal part on the *next* turn.

## Specific references from the diff

- `packages/core/src/utils/generateContentResponseUtilities.ts:89-117` — new partition loop: `inlineDataParts` is split into `unsupportedMimeTypes` (`mimeType.startsWith('audio/') || .startsWith('video/')`) and `filteredInlineDataParts`. When unsupported types exist, a steering message naming the offending MIME types and pointing at `@`-syntax is `unshift`ed onto `textParts`.
- `:131-143` — downstream uses now read `filteredInlineDataParts` everywhere `inlineDataParts` was previously read (the multimodal-FR-supported branch nests them, otherwise they're added as siblings; the empty-text fallback also counts only the filtered set).
- Test `packages/core/src/utils/generateContentResponseUtilities.test.ts:158-188` — `should filter out audio/video MIME types and add a steering message` asserts: zero `inlineData` parts in the result, the `output` field of the `functionResponse` contains `[SYSTEM ERROR: PROTOCOL_LIMITATION]`, both offending MIME strings (`audio/mpeg, video/mp4`), the `'@' syntax` instruction, and the original `'Some text'` is preserved.

## Verdict: `merge-after-nits`

The diagnosis matches the API contract, the fix is local to the conversion function, and the test asserts both the negative (binary parts gone) and positive (steering text present and pre-existing text preserved) directions. A few small nits keep it off `merge-as-is`.

## Nits / concerns

1. **MIME prefix list is hard-coded.** Hard-coding `audio/`, `video/` is the right call for today, but the underlying API rejection list could grow (e.g. specific niche types or future tightening). Lift the predicate into a small helper like `isUnsupportedFunctionResponseMime(mimeType)` and unit-test it directly so the list is grep-able and additive.
2. **Steering-message format is now load-bearing.** `[SYSTEM ERROR: PROTOCOL_LIMITATION]` has gone from prose into a structured token the agent's prompt is supposed to recognize. If anyone changes the wording later, agent behavior silently regresses. Either move the literal into a named constant in the same file (`UNSUPPORTED_MIME_STEERING_TEMPLATE`) and assert on the constant in the test, or document at the call site that the wording is part of the agent contract.
3. **Empty-output edge case.** When `textParts.length === 0` but unsupported types triggered the `unshift`, you've actually pushed the steering message *into* `textParts` — so the existing fallback at `:140-143` ("Binary content provided (N item(s))") will now never run for the audio/video case. Worth an extra assertion in the test that confirms this branch isn't accidentally double-counted (e.g. the `output` field is the steering message, not a "Binary content provided" string).
