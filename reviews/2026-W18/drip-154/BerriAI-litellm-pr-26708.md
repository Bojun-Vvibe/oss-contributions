# BerriAI/litellm #26708 — fix(proxy): auto-send user invitation emails when `send_invite_email is None` and email infra is configured

- PR: https://github.com/BerriAI/litellm/pull/26708
- Head SHA: (per `gh pr view` body)
- Files: `litellm/proxy/hooks/user_management_event_hooks.py` (+57), `tests/test_litellm/proxy/hooks/test_send_invite_email.py` (+170)
- Fixes: #20499

## Citations

- `user_management_event_hooks.py:30-58` — new `_is_email_sending_enabled() -> bool` first tries the v2 enterprise path (`from litellm_enterprise.enterprise_callbacks.send_emails.base_email import BaseEmailLogger`, then `litellm.logging_callback_manager.get_custom_loggers_for_type(BaseEmailLogger)`, returning True if any are initialized) — wrapped in `try/except ImportError` so OSS deployments without the enterprise extra don't crash. Falls back to the v0 alerting path: `if "email" in general_settings.get("alerting", []): return True`. Returns False otherwise. Docstring: "Returns True only if email is actually configured, preventing any email processing when the user has not opted in."
- `user_management_event_hooks.py:60-80` — new `_should_send_user_invitation_email(data, response) -> bool` is a pure function over the request/response shape, with a tri-state truth table for `data.send_invite_email`:
  - `True`  → send if `bool(response.user_email)` (regression test pins the no-email case as no-send)
  - `False` → never send
  - `None`  → auto-detect via `_is_email_sending_enabled()` AND `response.user_email`
- `user_management_event_hooks.py:158-194` — `async_send_user_invitation_email` rewired. Old code branched directly on `data.send_invite_email is True` (twice — once for v2 enterprise, once for v1 legacy), which meant the `None` default never sent. New code computes `should_send_email = _should_send_user_invitation_email(data, response)` once, threads it through both branches, and adds `enterprise_email_sent` flag so the v1 fallback only fires when v2 didn't (avoiding double-send when both are configured).
- `user_management_event_hooks.py:182-194` — v1 fallback condition flips from `if data.send_invite_email is True:` to `if should_send_email and not enterprise_email_sent:`. The `not enterprise_email_sent` guard is the load-bearing addition — without it, a deployment with both `email` in alerting AND a v2 enterprise email logger would send two emails per invite.
- `tests/test_send_invite_email.py:95-129` — new `test_v1_user_creation_no_email_when_send_invite_email_true_but_no_user_email`: passes `data.send_invite_email=True` but `response.user_email=None`, asserts `mock_slack_alerting.send_key_created_or_user_invited_email.assert_not_called()`. Pins the "explicit True but no email address" guard.
- `tests/test_send_invite_email.py:222-260` — new `test_v1_user_creation_sends_email_when_send_invite_email_none_and_email_configured`: passes `data` with `user_email="test@example.com"` and `data.send_invite_email` defaulted (None), `general_settings={"alerting": ["email"]}`, asserts the v1 send path was called once. This is the *new* behavior the PR enables.
- `tests/test_send_invite_email.py:263-` — companion negative `test_v1_user_creation_no_email_when_send_invite_email_none_and_email_not_configured` pins that without the alerting=["email"] config, the None default still doesn't send. Critical anti-regression — the previous behavior is preserved when email infra isn't set up.
- 7 total new tests; the test file mocks `litellm.logging_callback_manager.get_custom_loggers_for_type` and `litellm.proxy.proxy_server.general_settings` via `patch.dict(sys.modules, {...})` to keep the unit tests independent of the real proxy_server module.

## Verdict

`merge-after-nits`

## Reasoning

This fixes a real UX paper cut documented in #20499: deployments with email alerting configured (`general_settings.alerting = ["email"]`) and an enterprise email logger initialized still didn't send invitation emails by default because every callsite that creates a user had to *also* set `send_invite_email=True` explicitly. Users would configure email, create a user via the admin UI, and silently get no invite email — the only signal was the absence of the email itself. The fix routes through a tri-state evaluator that respects explicit True/False as overrides but does the right thing on the None default.

What the PR gets right:

1. **Pure function for the decision logic.** `_should_send_user_invitation_email` is a static method that takes only `data` and `response`, with no side effects. It can be unit-tested in isolation (the new test file does this), and the truth table is small enough to fit in the docstring. This is the correct shape for a policy function — every other production code path that needs the same decision in the future calls one method instead of re-implementing the truth table.

2. **The `not enterprise_email_sent` guard against double-send.** Old code's two `if data.send_invite_email is True:` branches were independent — a deployment with both v2 and v1 email infra configured (rare but legal) would have sent two emails per invite, and nobody would have noticed because the v1 path was guarded by feature-flag `use_enterprise_email_hooks` which silently set False on `ImportError`. The new code computes the decision once, sends through v2 if available, and *only* falls back to v1 if v2 didn't actually fire. That's the right priority order (v2 has richer templating) and the right guard.

3. **`try/except ImportError` for the v2 enterprise path.** OSS deployments without `litellm_enterprise` installed must not crash on import. The pattern is correct (narrow except on ImportError, swallow, fall through to v0 alerting check). Note the v0 check at `:55` reads `general_settings.get("alerting", [])` not `general_settings["alerting"]` — handles the case where `alerting` key is absent entirely.

4. **Negative tests pin the *previous* behavior as preserved.** `test_v1_user_creation_no_email_when_send_invite_email_none_and_email_not_configured` is the most important new test — it asserts that deployments without email infra configured still get the old behavior (no email). Without this test, a future refactor that "simplifies" the auto-detect to "always send unless False" would silently start spamming users at deployments that never opted in.

Nits worth raising:

1. **The behavior change is silent for existing deployments with email already configured.** Customers running LiteLLM in prod with `alerting: ["email"]` set and previously calling `/user/new` without `send_invite_email=True` will, on the version containing this PR, suddenly start sending emails to every newly-created user — including users created by automation, batch-import scripts, etc. that may have user_email populated for record-keeping but never intended an invite. Recommend a release-note callout: "**Behavior change:** when `send_invite_email` is omitted (default), invitations are now sent automatically if email infrastructure is configured. To preserve old behavior, set `send_invite_email=False` explicitly on all batch-create paths." This is the kind of change that surprises operators on Sunday morning.

2. **No structured signal that auto-send fired vs. explicit-send fired.** Telemetry / audit logs presumably record "invitation email sent" but not "this was sent because of the new auto-detect default vs. caller explicit True." Operators investigating "why did 500 users get emails this morning" would benefit from a structured field on the emit event (`auto_detected: true`). Out of scope, file as follow-up.

3. **The `enterprise_email_sent = True` flag is set inside a `for email_logger in initialized_email_loggers:` loop after the `await ...send_user_invitation_email(...)` call.** If one logger raises, the flag stays False and v1 fallback fires — which is probably the right behavior (best-effort delivery via the legacy path), but worth a docstring note. If the desired semantic is "v1 fallback only fires when no v2 logger was configured at all," the current code does the wrong thing on a partial v2 failure.

4. **`_is_email_sending_enabled()` does not log when it returns True/False.** Operators debugging "why didn't the email go out" would benefit from a `verbose_logger.debug(...)` line emitting which detection path triggered (v2 import OK + N loggers, v0 alerting list, neither). Cheap to add.

None of these block merge. Net: a small, well-tested fix to a real friction point, with the right backward-compat guards. Ship after a release-note callout in (1).
