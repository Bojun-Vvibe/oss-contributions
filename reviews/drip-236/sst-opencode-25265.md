# sst/opencode#25265 — fix(tui): gate logo subpixel rendering on truecolor support

- **PR**: https://github.com/sst/opencode/pull/25265
- **Head SHA**: `c8795a02651094b2f7b46db7f7a639756c16963d`
- **Size**: +4 / -1, 1 file
- **Verdict**: **merge-as-is**

## Context

Splash logo uses the upper-half-block trick (`▀`) so the `█` cell carries two independent colors — `fg` for the top "pixel" and `bg` for the bottom — letting `buildIdleState` shimmer at half-cell vertical resolution. On 256-color (indexed) terminals, `bg` is quantized to the nearest palette entry, which produces visible banding on the system-theme splash background. Related to #24872.

## What's right

**`packages/opencode/src/cli/cmd/tui/component/logo.tsx:1-2,557-558,688-689,792`** — three-line intervention:

1. New import `useRenderer` from `@opentui/solid`.
2. Hook usage at `:558` inside `Logo()` to obtain the renderer handle.
3. New memo at `:689`: `const useSubpixelBlocks = () => renderer.capabilities?.rgb === true`.
4. Predicate flip at `:792`: `if (char === "█")` → `if (char === "█" && useSubpixelBlocks())`.

When `useSubpixelBlocks()` is false, the `▀`-with-independent-colors branch is skipped and the existing fall-through path renders the glyph as a normal character (the subsequent renderer for `█` paths the cell through a single-color render). Net effect: truecolor terminals keep the smoother shimmer; indexed terminals get clean block glyphs without the 256-color quantization artifact.

The capability check via `renderer.capabilities?.rgb` is exactly the right boundary — it asks the renderer (which has already negotiated terminal capabilities) rather than probing `COLORTERM` directly, which would duplicate the detection logic and risk drift if the renderer's negotiation rules change.

## Risks / nits

- **`useSubpixelBlocks` is a function, not a memo, called inside the hot per-character render loop.** The body is a single property read so the cost is negligible, but if there's a profiler-visible overhead later the obvious fix is `const subpixelBlocks = createMemo(() => renderer.capabilities?.rgb === true)`. Cosmetic.
- **No test coverage.** Renderer-capability-gated rendering is hard to unit test without a mock terminal — defensible to skip given the surface (one predicate flip) and the visual nature of the regression.
- **Optional chaining `?.rgb === true` defaults to "indexed" if `capabilities` is missing.** That's the correct fail-safe direction (don't render subpixel on an unknown terminal).

## Verdict

**merge-as-is.** Surgical capability-gated rendering fix at the exact point where the truecolor-only assumption was load-bearing. The defaults fall on the correct side (indexed terminals lose a visual nicety rather than gain banding artifacts).
