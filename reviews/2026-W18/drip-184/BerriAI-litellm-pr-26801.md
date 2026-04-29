---
pr: BerriAI/litellm#26801
sha: 1d167974fff75d01313887a9c6dae9eb31bd9994
verdict: merge-as-is
reviewed_at: 2026-04-30T00:00:00Z
---

# fix(proxy): support SMTP_SSL (port 465) in send_email()

URL: https://github.com/BerriAI/litellm/pull/26801
Files: `litellm/proxy/utils.py`, `tests/test_litellm/proxy/test_utils.py`
Diff: 122+/7-

## Context

`send_email()` in `litellm/proxy/utils.py:4717` hardcoded
`smtplib.SMTP(host, port)` followed by an optional `starttls()` call —
that combination is the explicit-TLS upgrade pattern that only works on
port 587 (and a few other STARTTLS-capable ports). Production SMTP
relays that require *implicit* TLS on port 465 (Gmail, SendGrid, AWS
SES port 465, Office 365 legacy) need `smtplib.SMTP_SSL` from the very
first byte of the TCP connection. Previously, configuring
`SMTP_PORT=465` produced a hang followed by `SMTPServerDisconnected`
because the relay closed the plaintext connection before the
`STARTTLS` upgrade negotiation could start. Net effect: any deployment
whose corporate or compliance policy mandates port-465 implicit-SSL
SMTP couldn't send proxy emails at all.

## What's good

- Auto-detection at `utils.py:4720-4721`:
  `smtp_use_ssl = os.getenv("SMTP_USE_SSL", "False").lower() == "true"`
  then `use_ssl = smtp_use_ssl or smtp_port == 465`. The OR means port
  465 implies implicit SSL by default (matching the IANA-registered
  semantics) while the new `SMTP_USE_SSL` env override lets operators
  force implicit SSL on non-standard ports (legacy relays sometimes run
  SMTPS on 2525, 25025, etc.).
- Connection construction at `:4724-4727` is the smallest possible diff
  shape: pick `smtplib.SMTP_SSL(host, port)` if `use_ssl` else
  `smtplib.SMTP(host, port)`, both bound to `server_ctx`, then a single
  `with server_ctx as server:` block at `:4729` keeps the rest of the
  function untouched. No duplicated send-loop, no two-branch login
  block — the polymorphism happens at one site.
- Critical correctness detail at `:4730`: the existing `starttls()`
  call is now gated on `not use_ssl and os.getenv("SMTP_TLS", "True") !=
  "False"`. Calling `starttls()` on an `SMTP_SSL` connection is a
  protocol error (the channel is already encrypted; the relay returns
  `503 Bad sequence of commands`); this gate prevents the easy
  copy-paste mistake of "well it's TLS, of course we starttls". The
  prior `SMTP_TLS=False` opt-out path still works for the plain-587
  legacy case.
- Test coverage at `test_utils.py:23-152` is exhaustive across the
  state space:
  - `test_send_email_port_465_uses_smtp_ssl` (`:50-72`) — port 465 with
    `SMTP_USE_SSL` unset must call `smtplib.SMTP_SSL(host, 465)` and
    must NOT call `smtplib.SMTP` or `starttls`.
  - `test_send_email_smtp_use_ssl_env_forces_ssl` (`:75-97`) — port
    2525 with `SMTP_USE_SSL=True` must use `SMTP_SSL`, asserts the
    operator override path.
  - `test_send_email_smtp_use_ssl_env_lowercase` (`:100-122`) — locks
    the case-insensitive parse (`"true"` ≡ `"True"`), guarding against
    a future "strict" tightening that would silently break docker-compose
    files using lowercase env values.
  - `test_send_email_port_587_uses_starttls` (`:125-152`) — the
    backwards-compat regression guard: port 587, no `SMTP_USE_SSL`, must
    still call `smtplib.SMTP` + `starttls`. Critically asserts
    `mock_ssl.assert_not_called()` so a refactor that silently routes
    port-587 through `SMTP_SSL` would fail loud.
- Test fixture pattern (`MagicMock` with `__enter__`/`__exit__` for
  `with`-block compatibility, `monkeypatch.setenv` for env isolation,
  `monkeypatch.delenv(..., raising=False)` to handle inherited env in
  CI) is the right recipe — no fixture order coupling, no leaking env
  vars between tests.
- The `with server_ctx as server:` pattern at `:4729` correctly relies
  on both `SMTP` and `SMTP_SSL` implementing the context-manager
  protocol (they do, since CPython 3.0 and 3.0 respectively) so the
  cleanup-on-exception path is uniform.

## Verdict reasoning

Tightly-scoped fix that closes a real "feature configured but doesn't
work" gap (port 465 → hang). The auto-detect + explicit env override
pair is the right ergonomic shape — operators on standard 465 get it
for free, operators on non-standard SMTPS ports get an explicit knob.
Test matrix covers all four combinations of `(port=465 vs other) ×
(SMTP_USE_SSL set vs unset)` plus the case-sensitivity edge plus the
backwards-compat regression guard for port 587. Zero behaviour change
for existing port-587/STARTTLS deployments. The kind of fix that should
land same-day to unblock anyone who's been working around with a
sidecar relay.
