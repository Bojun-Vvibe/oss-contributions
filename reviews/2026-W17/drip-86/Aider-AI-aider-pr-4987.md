# Review — Aider-AI/aider#4987: fix: sanitize surrogate characters in messages before sending to LLM

- **Repo:** Aider-AI/aider
- **PR:** [#4987](https://github.com/Aider-AI/aider/pull/4987)
- **Head SHA:** `524b1290552991a262f01531ff8bccc7f068d533`
- **Author:** `graydeon`
- **Size:** +52 / -1 across 2 files
- **Verdict:** `merge-after-nits`

## Summary

Fixes #3460 — Windows builds with non-UTF-8 locales (e.g. Chinese GBK shells) emit surrogate code points (`\udcb0`) into file contents and console input. When `litellm`/`httpx` later tries to JSON-encode the outgoing request, the ASCII/UTF-8 encoding step throws `UnicodeEncodeError` and the entire request dies before reaching the provider. PR adds a recursive `sanitize_for_utf8(obj)` helper at `aider/models.py:1070-1085` and wires it in at `:1009` immediately before `kwargs["messages"] = messages` is consumed by `litellm.completion()`.

## Technical assessment

The fix is in the right place and uses the right primitive. The encode-with-`errors="replace"` then-decode round-trip at `models.py:1078` is the canonical "lossy but safe" approach for stripping unpaired surrogates — `errors="replace"` substitutes U+FFFD (`�`) for any byte that can't form a valid UTF-8 sequence, which is exactly the desired behavior because (a) the user already sees garbled text in the source so the LLM seeing `�` is no worse, (b) the request now serializes cleanly so the user gets a response instead of a crash, and (c) the substitution is reversible enough that a model can ask "what did you mean here?" if the corruption is meaningful.

The recursion at `:1080-1084` correctly walks `dict` values, `list` elements, and falls through unchanged for everything else (`int`, `bool`, `None`, `bytes`). One subtle correctness win: by checking `isinstance(obj, str)` first at `:1077`, then `dict`, then `list`, the function avoids treating `str` as iterable and recursing into characters — that ordering is load-bearing.

The 4 added tests at `tests/basic/test_models.py:560-589` cover (1) surrogate replacement in a flat string, (2) nested message-list-of-dicts with mixed clean and surrogate content (the realistic shape), (3) preservation of non-surrogate Unicode (`\u4e16\u754c` Chinese — important because users will rightly worry the function eats *all* non-ASCII), and (4) passthrough of `int`/`None`/`bool`. The nested test at `:570-577` does the right load-bearing thing — calls `json.dumps(result).encode("utf-8")` to assert the actual end-to-end contract (the request body must serialize), not just that strings change.

## Nits worth addressing pre-merge

1. **Tuple and bytes are not handled.** `sanitize_for_utf8((surrogate_str,))` returns the tuple unchanged. Same for `bytes`. Production message structures from `litellm` are dict-of-dict-of-list-of-str so this is unlikely to bite, but a one-line `isinstance(obj, tuple): return tuple(sanitize_for_utf8(x) for x in obj)` would close the surface. Bytes are trickier — a real bytes payload is probably an attachment that the user *wants* preserved verbatim.

2. **Performance on large messages.** Every `litellm.completion()` call now does a full deep walk of the message structure. For typical chat completions this is fine (a few hundred dicts) but for repo-map-heavy edits it's an O(n) pass over potentially megabytes of strings. The encode-then-decode round-trip is C-level fast in CPython but worth a quick benchmark on a realistic 100KB message. Consider gating with `if sys.platform == "win32" or os.environ.get("AIDER_FORCE_UTF8_SANITIZE"):` since this is a Windows-locale-only bug, sparing the 99% of users who never see the issue.

3. **Lossy without warning.** When sanitization actually fires, the user gets `�` characters in their conversation history with no log line indicating "we replaced N surrogate code points." A `if result != messages: logger.warning("sanitize_for_utf8 replaced N surrogate characters")` (counted via a side-channel) would help debug "why does the model see ◇ characters in my code."

4. **Missing test: surrogate in dict key.** `sanitize_for_utf8({"foo\udcb0": "bar"})` — currently only sanitizes values, not keys. Add a test pinning whether keys are also sanitized (probably should be, via `{sanitize_for_utf8(k): sanitize_for_utf8(v) for k, v in obj.items()}` at `:1081`).

## Verdict rationale

`merge-after-nits` because the bug is real, the fix is correct, and the test coverage is responsible. The four asks are low-effort: the Windows-only gating + log warning are the most valuable, the tuple/key coverage is housekeeping. None block merge — the current diff makes broken sessions work, which is a strict improvement over the status quo.
