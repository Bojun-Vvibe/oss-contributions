# browser-use/browser-use #4723 — security: verify init template integrity

- **PR:** https://github.com/browser-use/browser-use/pull/4723
- **Head SHA:** `0a7cd93cdd86d058b8f57e58ea7009399e170610`
- **Files changed:** 3 — `init_cmd.py` (+27/−36), `init_template_manifest.py` (+259/−0, new), `test_init_template_integrity.py` (+73/−0, new)

## Summary

Removes runtime fetching of `templates.json` from `github.com/browser-use/template-library` and replaces it with an in-package pinned manifest (`TRUSTED_INIT_TEMPLATES`, `TRUSTED_INIT_TEMPLATE_HASHES`). Every template file downloaded from GitHub is now verified against a SHA-256 hash pinned in `init_template_manifest.py` before being written to disk. Closes a real supply-chain hole: previously a compromise of the `template-library` repo (or any MITM on the raw.githubusercontent.com fetch) would have let arbitrary code into `browser_use init`-bootstrapped projects.

## Line-level call-outs

- `browser_use/init_cmd.py:48-58` — `_verify_template_bytes` raises `RuntimeError(f'No pinned integrity hash for template file: {file_path}')` when a file isn't in the manifest. Good fail-closed default. **But** `_fetch_template_bytes_from_github` at `:65-71` swallows this in `except (URLError, TimeoutError, Exception)` and returns `None`. **Catching bare `Exception` masks the integrity-failure signal** — a tampered file is reported to the caller identically to a network timeout, and the user sees "could not fetch template" instead of "template integrity check failed". The whole point of integrity checking is to *make tampering loud*. Either: (a) re-raise `RuntimeError` from the verify path and only swallow `URLError`/`TimeoutError`, or (b) propagate a distinct return type that the caller can surface to the user. As written, an attacker who can MITM the GitHub fetch gets a silent failure rather than an alarm.
- `browser_use/init_cmd.py:80-83` — `_fetch_binary_from_github` previously did its own `request.urlopen`; now it delegates to `_fetch_template_bytes_from_github`, so binary files are also verified. Good. But the `try`/`except` wrapper around a single delegated call is dead — `_fetch_template_bytes_from_github` already has its own broad except. Remove the outer wrapper or drop the inner one; having both means any exception is double-swallowed and harder to surface.
- `browser_use/init_template_manifest.py:1-259` (new file) — 259 lines of hand-maintained constants. **Operational risk:** every template-library update requires a coordinated PR here with the new SHA-256 values. If the `template-library` repo ships a fix and `browser-use` doesn't ship the matching manifest update, every `browser_use init` user will hit "integrity check failed" until the next browser-use release. Recommend at minimum: (a) document the bump procedure in the file's docstring, (b) ship a `tools/update_init_manifest.py` script that re-hashes the upstream and produces this file mechanically, (c) gate releases on the manifest matching upstream HEAD via CI.
- `browser_use/init_template_manifest.py` — uses `sha256:` prefix on hash strings (`f'sha256:{hashlib.sha256(content).hexdigest()}'` in `init_cmd.py:55`). Fine, but if the project ever wants to add SHA-3 / BLAKE2 / Subresource-Integrity-style multi-hash support, the prefix scheme is half-baked (no length, no separator standardization). Either commit to the SRI format (`sha256-<base64>`) which has a real spec, or document this as "internal format, not SRI".
- `test_init_template_integrity.py:1-73` (new file) — covers (1) trusted manifest is non-empty, (2) integrity mismatch raises. **Missing:** a test that the *production* download path (`_fetch_template_bytes_from_github`) actually invokes the verifier. The current tests call `_verify_template_bytes` directly, which proves the verifier works in isolation but not that it's wired into the fetch path. Mock `request.urlopen` to return tampered bytes and assert `_fetch_template_bytes_from_github` returns `None` (or, per the previous nit, raises). Without that, a future refactor that accidentally bypasses the verify call won't be caught.
- `test_init_template_integrity.py` is at the repo root, not under `tests/`. Inconsistent with the project's existing test layout — relocate to `tests/init/` or wherever the existing init tests live.

## Verdict

**request-changes**

## Rationale

The architectural move (pin manifest in-package, verify SHA-256 before write) is correct and high-value security work. But the implementation has one substantive bug — the broad `except Exception` swallows integrity failures into "network error" — that defeats the alarm property of the whole feature. An attacker who succeeds at MITM gets the same UX as a flaky network. Fix the exception handling so integrity failures are raised loudly, add a test that proves the wired-in verify path actually fires on tampered bytes, and move the test file under `tests/`. The rest is good. Don't ship a security feature whose primary failure mode is silence.
