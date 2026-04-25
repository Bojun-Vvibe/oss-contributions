# All-Hands-AI/OpenHands #14101 — fix(app): deliver pending messages queued during startup

- **Repo**: All-Hands-AI/OpenHands
- **PR**: [#14101](https://github.com/All-Hands-AI/OpenHands/pull/14101)
- **Head SHA**: `f2763217b5b6b7c6b770bdfdfb879d832a56ba73`
- **Author**: kayametehan
- **State**: OPEN (+80 / -11)
- **Verdict**: `merge-after-nits`

## Context

When a user enqueues a message before the agent's conversation
is `READY`, it gets stored under a `task-{taskId}` key and
later reassigned to `task-{conversationId}`. Bug: clients use
two different `task-{...}` encodings — `task-{uuid.hex}`
(no hyphens) and `task-{uuid-with-hyphens}` — and the server
only looked for the first. Messages queued under the
hyphenated form were never delivered.

## Design

`openhands/app_server/app_conversation/live_status_app_conversation_service.py:1369-1390`:

1. **Lines 1369-1372**: Replace single `task_id_str =
   f'task-{task_id.hex}'` with a list of two candidates —
   `[f'task-{task_id.hex}', f'task-{task_id}']`. Comment
   explicitly names both encodings, which is the right
   documentation move for a fix-by-list.

2. **Lines 1380-1386**: Loop over candidates, sum
   `updated_count` across both `update_conversation_id` calls.
   Order matters minimally because `update_conversation_id`
   should be a no-op for non-matching keys, so only one branch
   adds to the count in practice — but summing is correct for
   the (unlikely) case that messages were queued under both
   encodings.

3. **Logging** at lines 1376 and 1388 updated to print
   `task_ids` (plural) and the candidate list. Helpful for
   field debugging.

Tests:
- `test_live_status_app_conversation_service.py:2487-2528` —
  `test_process_pending_messages_updates_hyphenated_task_id`
  mocks `update_conversation_id` with `side_effect=[0, 1]`
  (first candidate misses, second hits) and asserts both calls
  were made *with the correct old-id strings*. Solid test;
  proves both encodings are tried and the sum logic works.
- `test_pending_message_service.py:288-305` —
  `test_update_conversation_id_accepts_hyphenated_task_id`
  proves the underlying SQL service is encoding-agnostic
  (which it is, since the key is just a string column).

## Risks

- **Hyphenated UUID matches loose patterns elsewhere.** If
  *any* other code path keys pending messages under
  `task-{uuid-with-hyphens}`, this fix silently sweeps them up
  too. That's probably fine — it's the same task — but a
  one-line note on which client encoding is preferred going
  forward would prevent the encoding split from getting wider.
- **Duplicate-update potential**: if a message is somehow
  queued under both encodings (e.g., client retried with
  different encoding mid-flight), `updated_count` would be
  2 and the message would be moved twice. The second call
  would no-op because the row's already at the new
  conversation_id, so this is benign — but worth a comment.
- **No deprecation path.** The right long-term fix is to
  pick one canonical encoding and migrate all clients to it;
  the comment at line 1369 should say "TODO: pick one
  encoding and remove the other" or similar, otherwise this
  list will grow a third entry the next time a client picks
  yet another shape.
- **Logging changes the format key from `task_id` to
  `task_ids`.** Any log-aggregator parsers looking for the
  exact `task_id=` pattern will need updating. Probably
  fine for an internal log line but worth flagging.

## Suggestions

- Add a `TODO` next to the candidates list pointing at the
  eventual single-encoding migration, or open a follow-up
  issue and link it.
- Consider short-circuiting: if the first call returns >0,
  skip the second — saves one round-trip on the (likely
  common) hex-encoding path. Trivial:
  `if updated_count == 0: try the hyphenated form`.
- Add the dual-encoding case to integration tests if any
  exist (the unit test mocks `update_conversation_id`
  directly, which proves the dispatch but not the SQL).

## What I learned

UUID encoding splits are a recurring failure mode at any
service boundary that round-trips IDs through string keys.
The correct durable fix is to canonicalize at ingress —
either parse to UUID and re-format on every read, or pick a
side. A defensive list-of-candidates is a fine *fix*, but it
should always carry a TODO for the canonicalization, or the
list grows by one every six months.
