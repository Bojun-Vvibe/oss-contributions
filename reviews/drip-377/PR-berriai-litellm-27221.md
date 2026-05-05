# BerriAI/litellm #27221 — fix(proxy): sort spend updates to prevent DB deadlocks

- Head SHA: `54d342da25445a41930f8029b5be7569d9de42c8`
- Diff: +165 / -26 across 3 files

## Findings

1. Real fix for a real production-class deadlock. `db_spend_update_writer.py` wraps each entity-type spend flush in a `prisma.tx(...)` `transaction.batch_()` (e.g. `:619-628`), and per-pod iteration order over `*_list_transactions: dict` previously came from Python's insertion-ordered dict — but two pods seeing different request orderings would acquire row-level locks in different orders and PostgreSQL would deadlock. The fix wraps every flush iteration in `sorted(...)` (user at `:624`, key at `:680`, team at `:721`, team_member at `:772`, org at `:813`, end_user at `utils.py:3215`, and the generic `_update_entity_spend_in_db` at `:892`). The accompanying inline comments correctly call out the load-bearing invariant: *"batch_() issues statements sequentially within the tx, so iteration order = lock acquisition order."*
2. Composite-key handling at `:771-776` is the subtlest part. The team_member key format `"team_id::<v>::user_id::<v>"` means lexicographic string sort is *equivalent to* sorting by `(team_id, user_id)` — which is what you want for consistent multi-row lock ordering. The inline comment documents this explicitly. Good.
3. The 134-line parametrized test at `tests/test_litellm/proxy/db/test_db_spend_update_writer.py:391-524` covers all 7 spend buckets (user, key, team, team_member, org, end_user, tag) with the same pattern: insert in a non-sorted order (`{"x_c": ..., "x_a": ..., "x_b": ...}`), assert the resulting `update_many` (or `upsert` for end_user) calls are in sorted-key order. This is the correct shape for a regression test — one parametrized spec generates 7 independent test cases with distinct IDs.
4. Concern (minor): the comment at `:622-625` says *"Sort by ID for consistent lock ordering across pods to prevent deadlocks"* but this only fixes deadlocks where the *contended row set* overlaps. If pod A is updating users `[a, b, c]` and pod B is updating `[b, c, d]`, sorted ordering gives both pods `b → c` for the contended subset and the deadlock is eliminated. If the row sets are disjoint there's no contention to begin with. So the fix is correct, but worth a one-line comment clarifying that this addresses *partial-overlap* contention, not the (impossible) case of disjoint sets.

## Verdict

**Verdict:** merge-as-is
