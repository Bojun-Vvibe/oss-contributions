---
pr: 10369
repo: cline/cline
sha: e6da0c47e053fdf2a09a2c147d1204e04a876daf
verdict: merge-after-nits
date: 2026-04-26
---

# cline/cline #10369 — fix(ollama): strip data URI prefix from images for Ollama API compatibility

- **Author**: alkul93123
- **Head SHA**: e6da0c47e053fdf2a09a2c147d1204e04a876daf
- **Link**: https://github.com/cline/cline/pull/10369
- **Size**: +20 / -10 in a single file: `src/core/api/transform/ollama-format.ts`. Closes #10368.

## Scope

The Ollama Chat API takes images as raw-base64 strings in a dedicated `images` field on the message, NOT as Data URIs in `content`. The previous code stitched `data:image/png;base64,…` into the text content, which Ollama either ignored or treated as garbage. Fix: extract image parts, strip the `data:<mime>;base64,` prefix if present, and put them in the `images` array. Replace the placeholder `(see following user message for image)` in `content`.

## Specific findings

- `ollama-format.ts:39-44` — tool-message branch: image parts inside a tool result are now pushed to `toolResultImages` after stripping the prefix via `replace(/^data:[^;]+;base64,/, "")`. The placeholder string in `content` is preserved. Note: the comment at line 39 says "Ollama SDK only supports tool results as a single string" — this PR's fix is consistent because the actual image data is now hoisted into the separate `images` field on the user message that's emitted alongside the tool block.
- `ollama-format.ts:47-58` (non-tool branch, the new bit): defines `nonToolImages: string[]` outside the `if (nonToolMessages.length > 0)` block at `:46`. The variable is in scope but only mutated inside the branch. Then attached as `images: nonToolImages.length > 0 ? nonToolImages : undefined`. Correctly omits the field when empty (Ollama doesn't choke on `undefined` but does prefer absence). Good.
- The regex `/^data:[^;]+;base64,/` is anchored, only matches the `data:<mime>;base64,` prefix, and is idempotent — applying it to already-raw base64 is a no-op. This handles mixed input gracefully (some sources may pre-strip, some may not).
- Edge case **not handled**: `data:` URIs can include parameters like `data:image/png;charset=utf-8;base64,…`. The regex `[^;]+` only captures the first token before the first `;`, so `image/png` matches; but if there are intermediate parameters (`image/png;charset=...;base64,`), the regex won't match and the prefix will leak through. Realistic? For images, almost never — `charset` doesn't apply. But defensive: `/^data:[^,]+;base64,/` would match the full mime+params shape. Worth tightening.
- Edge case **not handled**: non-base64 data URIs (`data:image/png,<urlencoded>`) and `data:` URIs without `;base64,` will fall through unchanged and Ollama will reject them. The previous code would also have produced garbage, so this is a no-regression but worth a future ticket.
- The tool-message branch and non-tool-message branch duplicate the strip-and-push logic. Two call sites, three lines each — borderline for a helper, but extracting `function stripDataUri(b64: string): string { ... }` would be cleaner and would centralize any future regex tightening.
- **No tests added**. The previous code did not have direct unit tests for the image path either, but this is a correctness fix with a known reproduction recipe (PR description gives one). Adding `convertToOllamaMessages-image.test.ts` with 3-4 cases (text-only, image with data URI, image with raw base64, mixed text+image) would close the regression risk for cheap. The repo presumably has Jest/Vitest setup.
- `ollama-format.ts:99` — unrelated `let content: string = ""` → `let content = ""` style cleanup. Harmless; could be split out but is one token.
- Diff is small and reviewable — 20 lines for a real bug fix is good.

## Risk

Low. Scope is one file, one transform function, one provider's image-encoding contract. Fail mode if the change is wrong is "Ollama still doesn't see images" — same as before the fix. Won't break non-image messages (no image path = no `images` field = unchanged behaviour).

## Verdict

**merge-after-nits** — tighten the regex from `[^;]+` to `[^,]+` to handle data URIs with parameters between mime and `;base64,`, extract a tiny `stripDataUri` helper to deduplicate the two strip sites, and add a unit test with at least the 3-4 cases above. Bug is real and fix is correct.
