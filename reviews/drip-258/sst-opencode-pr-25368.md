# sst/opencode PR #25368 — fix(transcript): wrap reasoning in `<thinking>` tags in markdown export

- PR: https://github.com/sst/opencode/pull/25368
- Head SHA: `f2a4965ab0714f160d89a36c598559fc54636039`
- Author: @1fanwang (Stefan Wang)
- Closes: #25308

## Summary

The markdown transcript exporter was rendering reasoning parts as `_Thinking:_\n\n<text>\n\n` — an opener with no closer, so a downstream parser had no reliable way to know where reasoning ended and the assistant response began. This PR replaces the wrapper in `formatPart`'s `reasoning` branch with `<thinking>...</thinking>`, matching what reasoning models (DeepSeek, Qwen3, GLM) emit natively and giving parsers a well-formed delimiter pair. The live TUI render in `cli/cmd/tui/routes/session/index.tsx` is intentionally untouched.

## Specific references from the diff

- `packages/opencode/src/cli/cmd/tui/util/transcript.ts:88-92` — single line swap from `_Thinking:_\n\n${part.text}\n\n` to `<thinking>\n${part.text}\n</thinking>\n\n`.
- `packages/opencode/test/cli/tui/transcript.test.ts:149-152` — the existing assertion `formats reasoning when thinking enabled` is updated to the new expected string. The `skips reasoning when thinking disabled` branch is untouched (still returns `""`).
- Author claims `bun test test/cli/tui/transcript.test.ts` passes 18/18.

## Verdict: `merge-after-nits`

Mechanically correct, behavior-preserving change with the test updated in lockstep. The only reason it's not pure `merge-as-is` is that `<thinking>` is also a literal HTML/JSX-ish token that some markdown renderers may either (a) silently drop or (b) try to interpret — worth being explicit about the choice.

## Nits / concerns

1. **Document the contract.** The PR body explains the rationale, but the chosen tag is now a stable export contract. Add a one-line comment above `formatPart`'s reasoning branch in `transcript.ts:88` saying "Use well-formed `<thinking>` delimiters so downstream parsers (and reasoning-native models) can round-trip reasoning blocks." Otherwise the next refactor will treat it as cosmetic.
2. **Markdown renderer interaction.** Some CommonMark renderers will treat `<thinking>` as an unknown raw HTML block and either pass it through verbatim, hide it, or error in strict modes. Worth a one-line note in the PR (or a follow-up doc) that the export is intended for parser consumption, not human Markdown rendering.
3. **No escaping of `</thinking>` inside `part.text`.** If a model literally writes the string `</thinking>` in its reasoning, the export now produces a malformed nested tag. Low-likelihood but trivial to defend against (e.g. `part.text.replaceAll("</thinking>", "<\\/thinking>")` or just call out the assumption in a comment).
