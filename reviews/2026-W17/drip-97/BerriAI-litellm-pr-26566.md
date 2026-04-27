# BerriAI/litellm#26566: fix(map_system_message_pt): handle list-typed content blocks

- **Author:** russellbrenner (russ)
- **HEAD SHA:** 76d12a22
- **Verdict:** merge-as-is

## Summary

A +20/-2 single-file fix in
`litellm/litellm_core_utils/prompt_templates/factory.py:107-130`
addressing a real `TypeError` crash on the message-merge path used
when an Anthropic-format request (e.g., from a Claude-SDK client
such as Claude Code) is routed to a provider with
`supports_system_message=False`. The previous one-liner
`next_m["content"] = m["content"] + " " + next_m["content"]`
crashed when either side held a list of typed content blocks
(`[{"type": "text", "text": "..."}]`) — string + list raises
`TypeError: can only concatenate str (not "list") to str` and
list + str raises a parallel error.

The fix is to type-aware: extract text from list-shaped system
content by joining `b["text"]` over `b["type"] == "text"` blocks,
then merge by either prepending a `{"type": "text", ...}` block
(when target content is a list) or string-concatenating (when
target content is a plain string). Behaviour is preserved for the
common all-string case, and the failing case from issue #23757 is
resolved.

## Specific feedback

- `factory.py:111-117` — `sys_text` extraction:
  ```python
  if isinstance(sys_content, list):
      sys_text = " ".join(
          b.get("text", "")
          for b in sys_content
          if isinstance(b, dict) and b.get("type") == "text"
      )
  else:
      sys_text = sys_content
  ```
  Correctly filters non-text blocks (Anthropic content blocks can
  also be `image`, `tool_use`, etc., though those rarely appear in
  system messages), guards against non-dict elements via
  `isinstance(b, dict)`, and uses `.get("text", "")` so a malformed
  text block contributes empty string instead of crashing. Solid.
- `factory.py:119-126` — the symmetric handling on the receiving
  side:
  ```python
  next_content = next_m["content"]
  if isinstance(next_content, list):
      next_m["content"] = [
          {"type": "text", "text": sys_text}
      ] + next_content
  else:
      next_m["content"] = sys_text + " " + next_content
  ```
  Prepending a fresh text block at index 0 is the right choice
  because system instructions typically should land before user
  content rather than after, and inserting at the head preserves
  the rest of the existing block list (including any image/tool
  blocks) untouched. The trailing-space delimiter `" "` in the
  string-string branch matches the pre-fix behaviour exactly,
  so no change for existing string-only callers.
- `factory.py:106-130` — the function still preserves the
  `next_role == "system"` branch downstream unchanged, which is
  correct because that path already creates a fresh user-typed
  message without string concatenation.
- The fix is appropriately narrow: it does not try to also handle
  the case where `sys_content` contains non-text blocks like
  `image` or `tool_use` (which would be highly unusual in a
  system message anyway), it just drops them silently via the
  `b.get("type") == "text"` filter. That's the right tradeoff —
  preserve text content, lose non-text content rather than crash.
- Linked issue: https://github.com/BerriAI/litellm/issues/23757
  is referenced in both the PR body and the inline comment, with
  the comment giving a one-line "why" pointer for the next
  reader. Good provenance hygiene.

## Risks / questions

- Test coverage: the PR's pre-submission checklist requires at
  least one test under `tests/test_litellm/`, and the file list
  shows only the source file. Strongly recommended to add a unit
  test that pins both branches: (1) list-typed system + string
  user → string concat, (2) list-typed system + list-typed user
  → prepended text block, (3) string system + string user →
  unchanged behaviour pinning. Not a blocker for the fix
  correctness (the diff is small and the logic is straightforward
  to eyeball), but the project's own contributing rule says it is.
  Recommending merge-as-is on the strength of the fix; a test
  follow-up PR is acceptable.
- Edge case: when `sys_content` is a list with zero text blocks
  (e.g., somehow only image content), `sys_text` becomes empty
  string, and the prepended text block becomes
  `{"type": "text", "text": ""}` — a degenerate but valid block.
  Some providers reject empty-text blocks. Worth either skipping
  the prepend when `sys_text == ""` or trimming. Non-blocking
  because the input shape is pathological.
- AI-attribution: the PR body marks this as AI-generated. Logic
  is small and verifiable; merge confidence is high based on
  inspection.
- The fix only addresses `map_system_message_pt`; if other
  template helpers in `factory.py` make the same string-concat
  assumption against list-typed content (e.g.,
  `_get_messages_for_completion`-style helpers), they will fail
  the same way. A repo grep for `m["content"] +` would surface
  any siblings worth fixing in a follow-up.
