# BerriAI/litellm #26957 — chore(guardrails): cover multimodal + Responses-API content shapes

- **PR**: https://github.com/BerriAI/litellm/pull/26957
- **Head SHA**: `d7677085712958f5d14e6b0ae5072600663afb4b`
- **Files reviewed**:
  - `litellm/proxy/guardrails/_content_utils.py` (NEW, +160) — shared helpers
  - `enterprise/enterprise_hooks/banned_keywords.py` (+5 -4)
  - `enterprise/enterprise_hooks/google_text_moderation.py` (+4 -5)
  - `enterprise/enterprise_hooks/openai_moderation.py` (+3 -5)
  - `enterprise/litellm_enterprise/enterprise_callbacks/secret_detection.py` (+20 -44)
  - `litellm/proxy/guardrails/guardrail_hooks/aim/aim.py` (+25 -11)
  - `litellm/proxy/guardrails/guardrail_hooks/ibm_guardrails/ibm_detector.py` (+75 -98)
  - `litellm/proxy/guardrails/guardrail_hooks/lakera_ai_v2.py` (+12 -10)
- **Date**: 2026-05-01 (drip-228)

## Context

Guardrail hooks across the codebase shared a copy-pasted pattern that
**only** inspected `data["messages"]` with `isinstance(content, str)`.
That meant three large categories of input were silently bypassing
every guardrail:

1. **Multimodal list-format content** — Chat Completions requests
   where `content` is a list of `{"type": "text", "text": "..."}` parts
   (the standard OpenAI/Anthropic shape for image+text).
2. **Responses-API requests** — these put user input under
   `data["input"]` (string OR list of messages OR list of content
   parts), not `data["messages"]`.
3. **Multi-completion responses** (Aim post-call hook only) — `n>1`
   completions; only `choices[0]` was inspected.

This is exactly the silent-skip class of bug that "fail open" hooks
specialize in producing.

## Diff walk

**`_content_utils.py:1-160`** — new shared module exposing four
functions that become the load-bearing source-of-truth for "what
counts as user-supplied text in a request body":

- `_iter_text_parts_in_content(content)` — handles the
  string/list-of-`{"type":"text"}` polymorphism in one place
- `_coerce_input_to_messages(input_value)` — handles the four shapes
  Responses-API `input` can take: bare string, list of full messages,
  list of strings, or content-parts; also defends against `None` /
  unknown shapes by returning `[]`
- `iter_user_text(data)` — yields every text fragment across both
  `messages` and `input`
- `walk_user_text(data, visit)` — **mutates** in place via a callback,
  used by secret-detection. Returns count of fragments visited so
  callers can early-exit
- `build_inspection_messages(data)` — synthesizes a chat-style flat
  `[{"role":..., "content": str}]` list for hooks that POST
  `{"messages": [...]}` to an external service (Aim, IBM, Lakera)

**`secret_detection.py:474-525`** — the most interesting site. The
old code had a particularly nasty rebind-loop bug at the
`data["prompt"]` list path (see `:504-516` of the old file): the loop
variable `item` was reassigned via `item = item.replace(...)` but
**never written back into the list**, so secret-redaction on
`data["prompt"]` was a no-op for the list-of-strings shape. The PR
fixes this with explicit index assignment at `:507`:

```python
for idx, item in enumerate(data["prompt"]):
    if isinstance(item, str):
        detected_secrets = self.scan_message_for_secrets(item)
        for secret in detected_secrets:
            item = item.replace(secret["value"], "[REDACTED]")
        data["prompt"][idx] = item   # ← the missing line
```

The PR adds an explicit comment naming the bug:

> Index back into the list — assigning to `item` would only rebind
> the loop variable and leave `data["prompt"]` carrying the
> unredacted secret.

That's the correct shape — pin the anti-pattern by comment so the
next maintainer can't strip it out as redundant.

**`aim.py:104, 165, 235-260`** — three call sites flipped:
- `:107` and `:166`: `data.get("messages", [])` → `build_inspection_messages(data)`
- `:235-260`: post-call hook now iterates **all** `choices` instead of
  just `choices[0]`, with concurrent inspection so multi-completion
  responses don't pay an n× latency penalty

The `n>1` fix is a real escape — the old code at `:236-243`
**returned the first choice's verdict for the entire response**,
meaning if the model produced 4 completions and only the first was
clean, the other 3 would be returned to the user without inspection.

## Observations

1. **The shared helper is the load-bearing artifact.** Every hook now
   imports the same `iter_user_text` / `walk_user_text` /
   `build_inspection_messages`. A future PR adding a fifth content
   shape (audio? function-call args? structured output?) only has to
   update `_content_utils.py` — every hook picks up coverage for free.
   That's exactly inverted from the prior state where each hook had
   its own slightly-different parser, none of which covered all
   shapes.

2. **`walk_user_text` correctly handles the mutate-in-place contract
   for Responses-API list-of-content-parts.** At
   `_content_utils.py:128-136`, the function uses `input_value[idx]
   = ...` for both string and dict cases, sidestepping the same
   rebind-loop bug that the old `secret_detection.py` had. The
   `{**item, "text": visit(item["text"])}` spread preserves
   non-`text` keys on the part dict, which matters for things like
   `cache_control` flags that vendors layer on top.

3. **The `aim.py` `n>1` fix is a real CVE-shaped escape.** Before:
   only `choices[0]` content was inspected via the output guardrail.
   After: every `Choices` instance is inspected concurrently via
   `asyncio.gather`. A guardrail that ships output verdicts based on
   `choices[0]` while letting `choices[1..n]` pass through is the
   exact shape of an "I told my model to give me 4 versions" bypass.

4. **Removed code in `secret_detection.py` was redundant, not lost
   coverage.** The old `data["input"]` handling at `:521-547` of
   the old file (string + list-of-strings + list-of-content-parts)
   is fully subsumed by the new `walk_user_text` call at `:489`.
   Reviewer can verify by reading the `walk_user_text` definition at
   `_content_utils.py:96-138` and confirming all three shapes are
   handled. They are.

5. **Nit: no test asserting the bare-string-over-int case.** The
   `_iter_text_parts_in_content` helper at `_content_utils.py:11-22`
   yields nothing if `content` is `None`, an int, a dict, or a
   list-of-not-dicts. That's the correct fail-safe behavior, but a
   regression test pinning `assert list(iter_user_text({"messages":
   [{"content": None}]})) == []` would lock the contract.

6. **Nit: `build_inspection_messages` joins multimodal text parts
   with `\n`.** That changes the wire shape posted to external
   guardrail APIs (Aim, IBM, Lakera) for multimodal content — an
   IBM/Aim policy that depended on per-part inspection would now see
   the parts concatenated. Worth a release-note line so operators
   running tightly-tuned policies don't get surprised.

## Verdict

**merge-after-nits** — closes a real silent-skip bug class
(multimodal + Responses-API + multi-completion bypass + secret
redaction list-rebind), centralizes the contract in one helper file,
documents the rebind-loop anti-pattern by inline comment. Add the
None/non-string regression test and a release-note about the
multimodal-join shape change posted to external guardrail APIs.

## What I learned

When the same parsing pattern is copy-pasted across 8 hooks, the bug
that's actually being shipped is "every hook covers a slightly
different subset of input shapes." The fix isn't to harden each hook
individually — it's to centralize the parser and let every hook share
coverage. The before/after dep graph is what makes this PR easy to
review: 8 hooks → 1 helper, every hook's input-extraction logic
shrinks 50-80%, and "did we miss a shape?" becomes a question about
one file instead of eight.
