# BerriAI/litellm#26464 — Harden team metadata handling in /team/new and /team/update

- PR: https://github.com/BerriAI/litellm/pull/26464
- Author: yuneng-berri (yuneng-jiang)
- +21 / -0
- Head SHA: `0a729388f147f7e3f79edd5bdd8e1bddcd1daac3`

## Summary

Strips a server-owned key (`team_member_budget_id`) from caller-supplied
`metadata` payloads on both `POST /team/new` and `POST /team/update`. Before
the fix, an authenticated caller could write any value at that key into team
metadata, and downstream budget logic that read the key back as a row pointer
would happily follow it — i.e., a confused-deputy hand-off where the caller
controls a server-trust boundary. The fix introduces a small denylist
(`SYSTEM_MANAGED_METADATA_KEYS`) and a `strip_system_managed_metadata_keys`
helper hung off `TeamMemberBudgetHandler`.

## Specific findings

- `litellm/proxy/management_endpoints/team_endpoints.py:162` (SHA
  `0a729388f147f7e3f79edd5bdd8e1bddcd1daac3`) — the denylist is a single-
  element tuple today: `("team_member_budget_id",)`. That's the only key
  currently known to be server-owned, but the structure is correct for adding
  more (e.g., any future cross-row pointer) without touching the call sites.
- `litellm/proxy/management_endpoints/team_endpoints.py:1055` — `new_team`
  calls the helper *after* the `_model_id` block but *before* `data.json()`
  serializes the payload to the DB write. Order is correct: the strip must
  happen before serialization, not after.
- `litellm/proxy/management_endpoints/team_endpoints.py:1747` — `update_team`
  strips on `updated_kv["metadata"]` (the post-`exclude_unset` dict), not on
  the raw `data` model. This is the right boundary because `updated_kv` is
  what actually flows into the DB write; mutating `data.metadata` would also
  work but would be more invasive. The `isinstance(updated_kv.get("metadata"),
  dict)` guard correctly no-ops when the field is absent or `None`.
- Helper at `team_endpoints.py:166`: `pop(key, None)` is idempotent and
  silent — i.e., a caller who *legitimately* needed to pass
  `team_member_budget_id` (there shouldn't be such a caller) would get no
  error feedback. Acceptable for a security-strip, but worth a debug log if
  the key was present and stripped, to help triage future "why didn't my
  metadata persist" tickets.

## Verdict

`merge-after-nits`

## Rationale

Correct, minimal, defense-in-depth fix on a real privilege-boundary problem.
Two small asks before merge: (1) add a `verbose_logger.debug(...)` inside
`strip_system_managed_metadata_keys` when a key was actually popped, so this
isn't a silent operation when triaging future incidents; (2) add a unit test
asserting that a caller payload `metadata={"team_member_budget_id":
"attacker-row", "extra_info": "ok"}` round-trips with `team_member_budget_id`
removed and `extra_info` preserved on both endpoints — the PR description
claims this was verified live but there's no checked-in test.
