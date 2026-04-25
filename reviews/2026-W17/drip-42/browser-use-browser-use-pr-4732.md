# browser-use/browser-use PR #4732 — feat: self-healing element recovery engine

- **URL:** https://github.com/browser-use/browser-use/pull/4732
- **Head SHA:** `e5a9c16ea050341ceab96065f3d35f48b5a9b11f`
- **Files touched:** 3 (`browser_use/dom/auto_heal.py` new, `browser_use/tools/service.py` integration, `tests/test_auto_heal.py` new)
- **Verdict:** `needs-discussion`

## Summary

Adds an opt-in (in practice: always-on, see below) self-healing
engine that fingerprints DOM elements before `click`/`input`, and on
"element index not found" failures attempts text-content, a11y label,
and role/tag-fallback recovery using `Runtime.evaluate` over a CDP
session.

## Specific references

- `browser_use/dom/auto_heal.py` (new, +383) — module docstring at
  lines 1-15 enumerates 5 strategies but the implementation only
  wires 3 (text, a11y, role/tag). `ElementFingerprint` dataclass
  captures `tag`, `text`, plus role/aria signals.
- `browser_use/tools/service.py` (+55 / −2) — integrates into both
  the click and input action handlers. Fingerprints before the action
  proceeds; on `IndexError`/lookup miss, calls `try_heal()` and
  returns a "retry" message after state refresh.
- `tests/test_auto_heal.py` (new, +190) — 10 tests covering
  initialization, fingerprinting, eviction-at-capacity, healing
  success/failure paths, stats, and confidence scoring.

## Reasoning

The engineering is solid (CDP `get_or_create_cdp_session` + JS
evaluation, graceful degradation, full stats), but this is a
behavioural change that needs maintainer discussion before merge:

1. **Silent retry semantics.** Auto-healing changes the agent's
   observed failure modes. An agent that previously got a hard
   "element gone" signal — and could decide to refresh state, scroll,
   or re-plan — now sometimes "succeeds" against a heuristically
   matched neighbor element. For a browser-automation library,
   correctness > convenience; merging this on by default could mask
   legitimate page-state bugs in user code.
2. **No opt-out flag visible** in the diff. PR description doesn't
   show a config switch on `BrowserSession`, `Tools`, or `Agent`.
   Should default to off, then ship on by default after a release of
   real-world data.
3. **Strategy count drift.** Docstring lists 5 strategies, code
   implements 3. Either trim the docstring or implement attribute-
   fingerprint and structural-position before merge.
4. **+800 token system-prompt cost** is mentioned in similar prior
   discussions of new tool actions; `heal_stats` adds another action
   surface — confirm this is wanted.

Recommend the author add a `auto_heal_enabled` flag (default `False`
for now) and update the docstring/strategy list. Then it's a
`merge-after-nits`.
