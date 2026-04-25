# browser-use/browser-use PR #4735 — Harden init template integrity and improve cloud tunnel recovery

- **URL:** https://github.com/browser-use/browser-use/pull/4735
- **Head SHA:** `cbfadfdbf9b9fdbb71e02a7ad28650b8b16d18af`
- **Files touched:** 14 (init/template integrity, browser session, watchdog, element redaction, CLI, plus 5 new test files)
- **Verdict:** `merge-after-nits`

## Summary

Three loosely-related hardening changes bundled:

1. Pin `browser-use init`'s template fetch to a fixed git SHA and
   verify SHA-256 of every fetched template file against an in-tree
   manifest.
2. Cloud tunnel recovery: on `net::ERR_TUNNEL_CONNECTION_FAILED`,
   stop the cloud browser, clear `cdp_url`, reprovision, retry
   navigation once.
3. Redact typed input from `Element.fill` debug logs to avoid leaking
   secrets/passwords into log files.

Plus a `subprocess.Popen` fallback for `LocalBrowserWatchdog` when
`asyncio.create_subprocess_exec` is unavailable.

## Specific references

- `browser_use/template_library_manifest.py` (new, +51) — declares
  `TEMPLATE_LIBRARY_COMMIT = '36bea3277f4c75b93a7c93b60c3cc4397e5aff77'`
  and `TEMPLATE_FILE_HASHES`. This is the integrity anchor consumed
  by `init_cmd.py` (+28 / −11).
- `browser_use/actor/element.py:fill()` — the typed value is replaced
  with `[redacted len=N]` (or similar) in the debug log line. Critical
  for users running with debug logging on a CI host.
- `browser_use/browser/session.py` (+55 / −11) — adds the tunnel
  recovery branch in the navigation error handler; clears `cdp_url`
  before reprovisioning so the next attempt can't reuse the dead
  endpoint.
- `tests/ci/test_init_cmd_template_integrity.py` (new, +81) and
  `test_cli_cloud_tunnel_recovery.py` (new, +78) — directly cover the
  two hardening paths.

## Reasoning

Each individual change is a clear net-positive: pinning the template
commit + hash-verifying fetched files closes a real supply-chain hole
on `init`; the tunnel recovery and the secret redaction are both
"obvious in hindsight" fixes with tests.

Nit / blocker-soft: this is three independent hardening items in one
PR (689 / 33). Reviewing as one is fine but bisecting will be
miserable if any one piece regresses. Suggest the maintainer either
(a) ask for it to be split into 3 PRs or (b) require that the three
test files keep passing in isolation. Author already notes
pre-commit didn't run cleanly on Windows — maintainer should verify
the hooks pass in CI before merge.
