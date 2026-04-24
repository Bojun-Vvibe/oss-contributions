# opencode PR #24150 — fix(transform): inject reasoning_content for ALL assistant msgs to fix DeepSeek thinking mode

Link: https://github.com/anomalyco/opencode/pull/24150

## What it changes

DeepSeek's "thinking" model (and any provider routed through
`@ai-sdk/openai-compatible` that surfaces `reasoning_content`) requires
the field to be echoed back on **every** assistant message in the
prompt history, including ones where the assistant produced no
reasoning text at all. The current `transform.ts` only injects it when
(a) the model has interleaved-reasoning capability *and* (b) the in-
memory message has non-empty `reasoningText`. That double gate misses:

- non-interleaved DeepSeek configs (most current `openai-compatible`
  setups);
- legacy DB-replayed messages that never recorded a reasoning part;
- string-content assistant messages from the older format.

The PR widens the trigger to `capabilities.reasoning` (drops the
interleaved requirement), defaults `providerOptions.reasoning_content`
to the literal string `"reasoning_content"` when missing, drops the
`if (reasoningText)` guard, and adds a string-content branch.

## Critique

The framing is right: DeepSeek's API treats `reasoning_content` as a
*structural* contract on assistant messages, not as optional metadata,
so silently dropping it on empty-reasoning turns produces the
observed "reasoning_content must be passed back" error mid-session.
The fix correctly identifies this as a contract divergence between the
upstream `ai-sdk` abstraction and the DeepSeek wire format.

Two concerns:

1. **Scope of the broadened trigger.** Dropping the interleaved-config
   gate means *every* `capabilities.reasoning` provider now gets a
   `reasoning_content` field defaulted in. That is fine for DeepSeek
   and for any OpenAI-compatible providers that ignore unknown fields,
   but a stricter provider could reject the unknown key. A safer shape
   is to gate on a per-provider opt-in (or on the `reasoning_content`
   `providerOptions` key being explicitly *configured*), and treat the
   string-default as a DeepSeek-shaped fallback. The PR's own
   `providerOptions.reasoning_content ?? "reasoning_content"` line is
   doing two jobs at once and obscures that.
2. **Coexistence with #24146.** The PR explicitly notes that #24146
   addresses the same symptom for interleaved models only. Landing
   both without coordination risks a double-injection or a
   precedence inversion if the interleaved branch overwrites the
   string-content branch. The PR should either supersede #24146 or
   add an assertion-style test that exercises the interleaved +
   non-interleaved + legacy-string matrix in one file.

## Suggestions

- Add a regression test that loads an old session JSON (string-content
  assistant message, no reasoning parts) and asserts the rendered
  prompt contains `reasoning_content` for *that* message.
- Replace the `?? "reasoning_content"` literal with a named constant
  on the provider-capabilities record so the contract is discoverable
  from the provider definition rather than buried in `transform.ts`.

Verdict: correctness fix in the right layer. Land after coordinating
with #24146 to pick a single owner of the field-injection logic.
