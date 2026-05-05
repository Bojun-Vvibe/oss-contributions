# BerriAI/litellm #27243 — fix(arize): stop O(L²) attribute emit by recording only the latest message in input_messages

- URL: https://github.com/BerriAI/litellm/pull/27243
- Head SHA: `28329cb1b69c3eb5bf6359d43f0a24b90ee2d2bd`
- Author: vanities
- Size: +17 / -13 in `litellm/integrations/arize/_utils.py`

## What it does

Replaces the per-span loop at `arize/_utils.py:42-51` that wrote `llm.input_messages.{0..N-1}.{role,content}` attributes for every prior message in a session with a single emit at index `last_idx = len(messages) - 1` carrying only the most recent turn's role+content. Cites internal ticket ENG2-1036 and the rationale "the most recent turn is sufficient for span-page detail; full history reconstruction belongs in DAL post-ingest."

## Observations

- **The O(L²) framing is correct for span-attribute volume across a session, not per-span.** Each span (=one LLM turn) previously wrote N attributes where N=conversation length, so a 50-turn conversation emits 50 + 49 + 48 + ... ≈ 1275 attribute writes via the Arize OTel pipeline. Across many sessions this multiplies the OTel exporter's encode/transmit cost meaningfully. Switching to 1 attribute per span turns the per-session total from `O(L²)` to `O(L)`.
- **Behavior change worth a release-note callout.** The Arize "input_messages" tab on the span page will now only show the *last* user/assistant message, not the full chat history that was previously available there. The PR justifies this with "full history reconstruction belongs in DAL post-ingest, not on every span" — which is a reasonable architectural stance but a real UX regression for anyone currently using the Arize UI's per-span inspect view to debug context.
- **Index-naming kept compatible.** Using `last_idx = len(messages) - 1` rather than `0` preserves the `llm.input_messages.{idx}.role`/`...content` schema position so that any downstream Arize parser keying off "index N is the Nth message" semantically still works. Good defensive choice — alternative `prefix = f"...0"` would have been simpler but might confuse consumers expecting index-aligned-with-position.
- The unmodified `last_message`/`role`/`content` extraction at `:34-38` (above the diff window) already exists for the standalone `LLM_INPUT_MESSAGES.{last_idx}` summary attribute, so the new code is structurally a deduplication of that same data path — there's no second `messages[-1]` lookup, both blocks read the same `last_message` local.
- Type/`Optional` propagation: `last_message.get("role")` and `.get("content", "")` mirror the surrounding code's defensive patterns; safe for messages with missing fields.

## Verdict: merge-after-nits

Real performance fix backed by a clear ticket reference, and the schema-position-preserving choice (`last_idx` vs `0`) shows attention to downstream compatibility. The one nit is the unmentioned UX regression for Arize span-page inspection — the PR description should explicitly call out "users will see only the last message in the input_messages tab; refer to DAL for full history" so support-channel owners aren't blindsided. A regression test asserting only one `LLM_INPUT_MESSAGES.{idx}.role` attribute is set per span (counter on `safe_set_attribute` in a fixture) would also be cheap insurance against accidental loop-reintroduction.