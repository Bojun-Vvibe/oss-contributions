# BerriAI/litellm PR #26546 — feat(guardrails): add peyeeye PII redaction & rehydration guardrail

- **PR:** https://github.com/BerriAI/litellm/pull/26546
- **Author:** tim-peyeeye
- **Head SHA:** `f58724611efffa8f25da63c86c8db85586c64874`
- **Files:** 5 (+760 / -0)
- **Verdict:** `request-changes`

## What it does

Adds a built-in `peyeeye` guardrail wired into the litellm proxy's
guardrail-hooks system. Two-phase flow: pre-call hook calls
`https://api.peyeeye.ai/v1/redact` to swap PII tokens out of every
`messages[].content` text part, caches the returned session id under the
request's `litellm_call_id`, then post-call hook hits a rehydrate endpoint
to swap originals back into `response.choices[].message.content`.

Two session modes:
- **stateful** (default) — peyeeye holds the token→value mapping under a
  `ses_…` id; rehydrate references the id.
- **stateless** — peyeeye returns a sealed `skey_…` blob; nothing
  retained server-side.

## Specific reads

- `litellm/proxy/guardrails/guardrail_hooks/peyeeye/__init__.py:1-36` —
  standard initializer pattern matching other built-in guardrails. Reads
  config from `litellm_params.peyeeye_locale`, `peyeeye_entities`,
  `peyeeye_session_mode`. Two registries, `guardrail_initializer_registry`
  and `guardrail_class_registry`, both keyed by
  `SupportedGuardrailIntegrations.PEYEEYE.value`. **Need to verify** the
  enum entry was added in `litellm/types/guardrails.py` (not in this
  diff). If the enum value is missing the import will fail at module load
  time and the guardrail won't register.
- `peyeeye.py:18` — `DEFAULT_API_BASE = "https://api.peyeeye.ai"`. Plain
  string, no allow-list check. If a misconfigured `api_base` is provided,
  the guardrail will happily POST every prompt to it — including the very
  PII it's supposed to redact. Should at minimum log a warning when
  `api_base` differs from the default, or require an explicit
  `peyeeye_allow_custom_api_base: true` to opt in.
- `peyeeye.py:62-69` — `peyeeye_api_key` resolution falls back to
  `os.environ.get("PEYEEYE_API_KEY")`. If unset, raises
  `PEyeEyeGuardrailMissingSecrets` at construction time — **good, fails
  loud**.
- `peyeeye.py:104-145` — `async_pre_call_hook`. Iterates messages via
  `_iter_message_text(messages)`, batches all text parts, calls
  `_redact_batch`, then re-injects redacted strings back into the
  message structure. Batches in a single API call, which is the right
  shape for latency. **However**, the response-length check is only
  `len(redacted_texts) != len(text_parts)` — that's a count mismatch but
  not an *order* check. If the peyeeye API ever returns out-of-order
  results (or partial results with the same count), every message would
  get the wrong redaction injected without the guardrail noticing. A
  per-text round-trip token (UUID per text_part, echoed in the response)
  would close that hole.
- The `litellm_call_id` cache TTL is `SESSION_CACHE_TTL_SECONDS = 3600`
  (1 hour). For long-running agent sessions where pre-call and post-call
  are separated by tool execution, this is fine. For very long tool
  loops (>1h, possible for sandboxed shell + slow remote builds) the
  cached session id will expire and rehydration will silently fail. The
  post-call hook should detect a missing cache entry and *fail loud*
  (return the prompt's original or raise) rather than return the
  unrehydrated response with `[REDACTED-PII-7af3…]` placeholders to the
  user.
- `peyeeye.py` is 365 lines but the diff only shows ~150 of them. The
  bulk I can't see includes `_redact_batch`, `_rehydrate`,
  `_iter_message_text`, and the post-call hook itself. Several risk
  factors below depend on those.

## Risk surface

**Medium-high** — this is a new outbound network dependency on every
prompt request and on every model response. Failure modes:

1. **peyeeye API down or slow.** No timeout is visible in the diff. Need
   `httpx.Timeout(connect=2, read=10)` or similar — without it, a slow
   peyeeye API stalls every chat completion.
2. **peyeeye API leaks raw prompt to logs / outage / breach.** This
   guardrail's whole value prop is "PII never reaches the LLM," but it
   *also* means PII reaches `api.peyeeye.ai`. That's a one-vendor-trust
   reduction, not a zero-trust improvement. Worth documenting in the
   user-facing docs (the `docs/` portion of this PR isn't visible in the
   diff stat — assuming it exists).
3. **Tool-call args & assistant message content not addressed in the
   visible code.** PII could ride in an assistant's tool-call arguments,
   not just in user `content`. If `_iter_message_text` only walks string
   `content`, anything in `tool_calls[*].function.arguments` is leaked
   in cleartext to the model. Need to verify scope.
4. **Streaming responses.** Post-call hook signature implies a complete
   response. If the proxy is in streaming mode, rehydration of partial
   chunks isn't possible without buffering — and buffering breaks the
   point of streaming. The PR description doesn't address streaming
   mode.
5. **No retry on transient peyeeye failure.** A flaky network blip
   between proxy and peyeeye now means either the request fails
   (default_on=true) or the unredacted prompt goes through (default_on
   not set / fail-open). Both are bad — need explicit retry+timeout
   policy.
6. **Stateful session leak.** In `stateful` mode peyeeye is holding
   a token→PII mapping. If the post-call hook never fires (proxy
   crashes, request canceled mid-flight), the session entry is leaked
   on peyeeye's side until it TTLs out. The PR mentions session cleanup
   but I can't see the cleanup path in the visible diff.

## Suggestions before merge

1. **Add timeouts** on every `async_handler` call. Default to 10s
   read / 2s connect. Make configurable.
2. **Verify and add** the `SupportedGuardrailIntegrations.PEYEEYE` enum
   entry — otherwise the import fails silently.
3. **Add an outbound-host allow-list check** on `api_base`, or at least
   a `WARN` log when the user overrides the default.
4. **Document the streaming-mode behavior** explicitly — does the
   guardrail force `stream=False`, buffer-then-rehydrate, or skip
   rehydration on streamed responses?
5. **Per-text round-trip token** for ordering safety in `_redact_batch`.
6. **Cache TTL on cache miss in post-call** — fail loud (raise),
   don't return placeholder strings to the caller.
7. **Add a dedicated test fixture** for tool-call argument PII to verify
   scope.
8. **Document the trust reduction** in the guardrail docs page: "PII no
   longer reaches the LLM, but it does reach api.peyeeye.ai. If your
   threat model is 'never let PII leave my proxy host,' use Presidio
   instead."

Verdict: request-changes — the architecture is sound and the
redact→LLM→rehydrate roundtrip is the right shape, but the visible code
ships without timeouts, with weak ordering guarantees, and without an
explicit policy for streaming mode and post-call cache misses. These
are the exact failure modes you'll hit in production within the first
week, and they're all fixable in this PR before merge.

## What I learned

A "redact before send, rehydrate after" PII guardrail is a clever
sleight-of-hand — the LLM never sees raw PII so the model provider
never logs it — but it transfers the trust boundary, it doesn't
eliminate it. The guardrail vendor now sees every prompt and response
in the clear. Whether that's a net win depends on which vendor you
trust more. The technical pitfalls (timeouts, ordering, cache TTL,
streaming) are all the same pitfalls of any "two-phase async middleware
sharing state via a key" — none unique to PII redaction, but all of them
critical when the guardrail's job is to be invisible-when-working and
loud-when-broken.
