# cline/cline PR #10380 — [Aikido] Fix security issue in dompurify via minor version upgrade from 3.3.3 to 3.4.1 in webview-ui

- **URL:** https://github.com/cline/cline/pull/10380
- **Head SHA:** `5b7beaefeb4a90099364a49d42e7270f7c9ea1f6`
- **Files touched:** 2 (`webview-ui/package.json`, `webview-ui/package-lock.json`)
- **Verdict:** `merge-as-is`

## Summary

Defensive dependency bump driven by Aikido scanner. Lifts the
HTML-sanitization library `dompurify` from `^3.3.3` → `^3.4.1` in the
webview-ui package only. Patch-level upgrade within the same major,
no API breaks expected.

## Specific references

- `webview-ui/package.json` — single line: `"dompurify": "^3.3.3"` →
  `"dompurify": "^3.4.1"`.
- `webview-ui/package-lock.json` — corresponding lockfile delta (+39 /
  −16 lines) covering the resolved version, integrity hash, and the
  transitive dedupe.

## Reasoning

Sanitizer upgrades for the renderer surface are the cheapest wins in
an Electron/webview context — `dompurify` is exactly the boundary you
want patched promptly. 3.3.3 → 3.4.1 is a minor bump; the changelog
does not introduce config-shape changes that would require call-site
updates, and the only consumer surface in cline's webview is the
markdown rendering path which uses default sanitizer config.

No source files changed, lockfile is consistent, no transitive risk
beyond `dompurify` itself. Merge as-is once CI is green; nothing to
gate on.
