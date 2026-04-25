# browser-use/browser-use PR #4736 — fix(element): redact sensitive values in fill() debug logging

- **URL:** https://github.com/browser-use/browser-use/pull/4736
- **Head SHA:** `99e9a455e77b9809c9358674555ac2ab87133b18`
- **Files touched:** 2 (`browser_use/actor/element.py`, `browser_use/llm/schema.py`)
- **Verdict:** `merge-as-is`

## Summary

Stops `fill()` from dumping the typed string (passwords, tokens, MFA
codes) into DEBUG logs. Replaces the value with `[REDACTED N chars]` so
operators can still see typing happened and roughly how much. Also
removes a dead `'type'` entry from a schema-validation key list.

## Specific references

- `browser_use/actor/element.py:421` — single-line change:
  `logger.debug(f'Typing text character by character: "{value}"')`
  becomes `... "[REDACTED {len(value)} chars]"`. Preserves observability
  (length, the fact that typing started) without exposing the secret.
- `browser_use/llm/schema.py:91-92` — drops `'type'` from the
  `optimize_schema` validation-fields list. Per the PR body and a quick
  read of the surrounding code, `'type'` is already handled in an
  earlier branch, so this entry was unreachable. No behaviour change
  expected.

## Reasoning

This is exactly the kind of small, defensive PR that should land
immediately. The redaction matches the convention used elsewhere in the
codebase (length-preserving placeholder), and the schema cleanup is a
genuine dead-code removal with the rationale documented.

No tests are added, but for a one-line log-format change that's
proportional. A future hardening pass could add an opt-in
`BROWSER_USE_LOG_FILL_LENGTH=0` env to also suppress the length, but
that's a separate change.

Merge as-is.
