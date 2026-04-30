# google-gemini/gemini-cli #26238 — Fix topic marker leakage in CLI output

- **URL:** https://github.com/google-gemini/gemini-cli/pull/26238
- **Head SHA:** `7c0603ce0a76c343eddd0f061f9cfcf8422d23e1`
- **Files:** `packages/core/src/prompts/{promptProvider,promptProvider.test,snippets,snippets.legacy}.ts` (+6/-4)
- **Verdict:** `merge-after-nits`

## What changed

The model was leaking `[Active Topic: ...]` markers into chat output (i.e. echoing the system-prompt-injected active-topic context line back at the user). Two coordinated fixes:

1. **Marker shape change.** `promptProvider.ts:280-285` switches the active-topic injection from a square-bracket `[Active Topic: ...]` shape to an XML-tag `<active_topic>\n${sanitizedTopic}\n</active_topic>` shape. Sanitization regex correspondingly broadened from `.replace(/\]/g, '')` to `.replace(/[\[\]<>]/g, '')` to strip both bracket *and* angle-bracket characters from user-supplied topic text (defense against tag-injection in the topic itself).
2. **Prompt-side instruction.** Both `snippets.ts:659` and `snippets.legacy.ts:530` add a `**No topic markers in chat:**` bullet to the `## Topic Updates` section telling the model to never echo `[active topic]` or `<active_topic>` markers in chat responses, with a clear "communicate exclusively via the ${UPDATE_TOPIC_TOOL_NAME} tool" mandate.
3. Test at `promptProvider.test.ts:361,372` updates the inclusion-assertion strings to the new XML shape.

## Why it's right

- The XML-tag shape is the right call. Most production assistant prompts now use `<tag>...</tag>` for context blocks (Anthropic, OpenAI structured prompting examples) precisely because models are trained to recognize them as "system-injected, don't echo back" boundaries; square-bracket markers are far more likely to be re-emitted as natural-language formatting (because they look like markdown footnote refs / abbreviations).
- The sanitization regex broadening at `:283` is necessary correctness — without stripping `<` and `>` from the topic string, a topic literally named `<active_topic>injected</active_topic>` could close the tag early and inject content. Defense-in-depth on a small-cost regex.
- Belt-and-suspenders: applying the prompt-side instruction in *both* `snippets.ts` and `snippets.legacy.ts` guarantees coverage regardless of which prompt builder a given session resolves to. The matching test snapshot update locks the new inclusion shape.
- The instruction wording at `:659` is conservatively scoped: "in your chat responses" + "communicated exclusively via the ${UPDATE_TOPIC_TOOL_NAME} tool" — leaves room for the model to reference topics indirectly ("I'm working on the parser cleanup task") without forbidding all topic-related natural language.

## Nits

- **Sanitization is character-stripping, not escaping.** A topic named `Refactor <auth>` becomes `Refactor auth` post-sanitize, which is *information loss*. Operators won't notice this for most topics, but for technical project names containing `<>` (e.g. C++ template names, generic types) the displayed topic in the prompt will silently differ from what the user typed. A `.replace(/[\[\]<>]/g, c => ({"<":"&lt;",">":"&gt;","[":"\\[","]":"\\]"}[c]))` style escape would preserve the information at the cost of one regex callback. Not blocking, but worth a comment explaining the trade-off.
- **Negative regression test missing.** The PR has positive assertions ("output contains `<active_topic>...</active_topic>`") but no test that asserts the model's *output* (or a render of it) doesn't contain `[Active Topic:`. That's the actual user-facing bug. Even a unit test that runs `getCoreSystemPrompt(...)` and asserts `prompt` contains the new marker AND does NOT contain `[Active Topic:` would lock the migration. The current test only flips the inclusion string.
- **`<active_topic>` shape doesn't match the surrounding prompt convention.** Run `grep -n '<[a-z_]*>' packages/core/src/prompts/snippets.ts` — if there are existing tag conventions like `<context>` or `<system_note>`, this should match (e.g. `<active_topic>...` vs `<context type="active_topic">...`). Worth a one-line audit to keep the prompt taxonomy coherent for future maintainers.
- **The PR title leaks information that suggests a broader scope ("topic marker leakage in CLI output")** but the diff doesn't actually verify any *output*-side scrub — it relies entirely on the prompt instruction. If the model still occasionally emits the marker (which it will, sometimes — prompt instructions are not hard contracts), the bug recurs. A post-hoc output-side regex scrub (`response.replace(/<\/?active_topic>[^<]*<\/active_topic>/gi, '')` or similar) at the rendering layer would be a stronger guarantee. Worth a follow-up.

## Risk

Low. The marker shape change is a one-shot migration; any in-flight session with the old marker in its history will simply have a stale-shape line, no functional break. The sanitization broadening is strictly more conservative (strips more chars). The prompt-side instruction is additive and doesn't conflict with any existing rule.
