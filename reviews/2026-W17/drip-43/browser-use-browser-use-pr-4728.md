# browser-use/browser-use PR #4728 — feat(cli): add --proxy-url flag for local Chromium sessions

- **URL:** https://github.com/browser-use/browser-use/pull/4728
- **Author:** @vevshy1
- **State:** OPEN (target `main`)
- **Head SHA:** `e084aae3a8ff30ea3d5482428bcb396f0abdcb88`
- **Files:** `browser_use/skill_cli/daemon.py`, `browser_use/skill_cli/main.py`

## Summary of change

Adds `--proxy-url`, `--proxy-bypass`, `--proxy-username`,
`--proxy-password` CLI flags that flow through to the local-Chromium
Daemon. Also tweaks the daemon `ping` reply to expose both the
configured (`cdp_url`) and live (`live_cdp_url`) endpoints separately
so the config-mismatch detection in `ensure_daemon` doesn't ping-pong
restart whenever Chromium picks a different debugger port.

## Findings against the diff

- **`daemon.py` L41–60:** four new constructor params stored on
  `self.proxy_*`. Good — straight-through plumbing.
- **`daemon.py` L135–141:** params forwarded into
  `_get_or_create_session(...)`. The `BrowserSession` constructor
  must accept those kwargs; the diff doesn't show that side, so a
  reviewer should confirm `BrowserSession.__init__` already plumbs
  proxy fields to the underlying Playwright/CDP launcher (it does
  in this repo as of recent versions, so likely fine).
- **`daemon.py` L288–315 (ping):**
  ```diff
  -            'cdp_url': live_cdp_url,
  +            'cdp_url': self.cdp_url,
  +            'live_cdp_url': live_cdp_url,
  ```
  This is a **subtle behavior change** worth calling out: existing
  external consumers of `ping`'s `cdp_url` field now receive the
  configured value (often `None` for local sessions) instead of the
  live one. Internal caller `_handle_sessions` is updated to fall
  back via `data.get('live_cdp_url') or data.get('cdp_url')`, but
  any third-party tooling parsing `ping` would silently break. A
  brief CHANGELOG note covers it.
- **`daemon.py` L309–315 (proxy fields in ping):**
  ```python
  'proxy_url': self.proxy_url,
  'proxy_bypass': self.proxy_bypass,
  'proxy_username': self.proxy_username,
  # Do not return proxy_password — still include its presence
  'proxy_password_set': bool(self.proxy_password),
  ```
  ✅ Correct: password redacted, presence-flag exposed for the
  config-mismatch comparison. **Username is still leaked over the
  socket** — debatable; usernames aren't usually secrets but if the
  socket is shared across users on a host that's a small infoleak.
  Consider redacting it too or adding a docstring note.
- **`main.py` L445–453:** the `data.get('proxy_password_set', False)
  == bool(proxy_password)` comparison correctly avoids restart
  flapping when the password is set.
- **`main.py` L1481+:** `--proxy-*` flags added to
  `explicit_config` detection list. ✅ Forces a fresh daemon when
  the user changes proxy intent.
- **Argument parser duplication:** flags are defined twice — once on
  the top-level `build_parser()` and again on the daemon's own
  parser. Necessary because the daemon is a separate subprocess,
  but the help text duplicates verbatim — fine.

## Verdict

**merge-after-nits**

Solid feature, properly threads four new params through CLI →
ensure_daemon → daemon subprocess → BrowserSession with sensible
mismatch detection. Three nits to address before merge:

1. **Add a CHANGELOG/release note** about the `cdp_url` field in
   `ping` now being the configured value (with `live_cdp_url` as
   the new live source).
2. **Consider redacting `proxy_username`** in the `ping` reply, or
   document why it's intentionally exposed.
3. **Add a smoke test** (or at least a manual-test script) covering
   `--proxy-url socks5://...` since the failure mode (proxy never
   wired into Playwright) would otherwise only surface to end users.
