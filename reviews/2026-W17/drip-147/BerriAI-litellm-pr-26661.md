# BerriAI/litellm #26661 — Add gateway-managed conversation IDs for the Responses API

- PR: https://github.com/BerriAI/litellm/pull/26661
- Head SHA: `0e38dfc2575e7a3cf6896feae437a248abfc593c`
- State: MERGED
- Author: krrish-berri-2 (Krrish Dholakia)
- Files: `litellm/responses/utils.py` (+68/−0), `litellm/responses/main.py` (+102/−0), `litellm/responses/litellm_completion_transformation/session_handler.py` (+47/−0), `tests/openai_endpoints_tests/test_e2e_litellm_conversation_id.py` (+238/−0)

## Observations

- Feature shape: when a `/v1/responses` request has `conversation` prefixed with `litellm_conv_id_`, the proxy claims it (rather than passing through to the upstream provider). It (1) sets `LiteLLM_SpendLogs.session_id` to the conversation id, (2) replays prior turns from a `DualCache` (in-memory + Redis when configured) with a spend-logs DB fallback on cache miss, and (3) strips the synthetic id from the body before forwarding upstream. Foreign-prefixed ids (anything not `litellm_conv_id_*`) pass through untouched.
- **The design choice that makes this PR good**: no new tables, no new migrations, no new endpoints. Reuses the already-indexed `LiteLLM_SpendLogs.session_id` column and the existing `/spend/logs/session/ui?session_id=...` route. This is the right shape for a feature in a system where schema migrations are a deployment risk — the new feature *infers* state from existing telemetry rather than introducing a new persistence surface. Same shape as the `previous_response_id` flow it parallels.
- `LITELLM_CONVERSATION_ID_PREFIX` constant in `responses/utils.py:1-68` gives a single source of truth for the discriminator. The `_apply_litellm_managed_conversation` pre-call hook + `_record_litellm_managed_conversation` post-call writer pair in `responses/main.py:1-102` is the right symmetric shape — read on entry, write on exit, both gated by the same prefix check.
- `DualCache` (in-memory + Redis lazy init) handles the race between async spend-logs flush (~12s in dev) and back-to-back agentic-loop turns. Without the cache, the second turn in a tight loop would hit empty spend logs and lose history. PR is *honest* about the tradeoff: "Multi-worker without Redis is best-effort. Falls back to spend-logs DB on cross-worker miss, which then races with the ~12s async flush. Configure Redis to make multi-worker correct." That's exactly the right disclaimer to ship.
- Test surface (`test_e2e_litellm_conversation_id.py:1-238`): three E2E cells against a live proxy:
  - `test_litellm_conversation_id_persists_across_responses_calls` — both calls land in spend logs with the conv id as `session_id`, distinct provider response ids, retrievable via `/spend/logs/session/ui`. Tests the persistence invariant.
  - `test_litellm_conversation_id_history_is_replayed_to_model` — secret token planted in turn 1 is recalled in turn 2. This is the **load-bearing test** — proves replay actually reaches the model, not just that history was stored.
  - `test_non_litellm_conversation_id_is_not_intercepted` — foreign-prefixed ids pass through. Closes the "we don't break upstream Responses API conversations" risk.
- Plus a unit smoke for the `DualCache` Redis branch verifying cross-worker reads return Redis value after wiping in-memory layer.

## Risks / nits

- "Async-only" limitation: the hook lives in `aresponses`, not sync `litellm.responses(...)`. PR documents this honestly. Risk is users who somehow exercise the sync path silently get no replay. Worth a `warnings.warn` in the sync entry point if `conversation` starts with the prefix.
- "No per-key auth scoping" — the conv id is a global namespace. Anyone holding a conv id can resume the conversation regardless of the API key that originated it. Same posture as `previous_response_id` today, but: this is a tightening opportunity for *both* flows in a follow-up. Worth a CVE-class threat-model section in the PR body acknowledging the cross-tenant risk explicitly.
- `LITELLM_CONVERSATION_ID_PREFIX = "litellm_conv_id_"` — fine namespace, but the prefix is a public protocol contract now. Future versioning path needs thinking — `litellm_conv_id_v2_` etc.
- `session_handler.get_chat_completion_message_history_for_session_id` — the new method (+47 lines) reads via Prisma `find_many`. PR cites "CLAUDE.md guidance" for using `find_many` — verify the existing project convention actually says this; if not, rename the cite.
- No test for the "cache populated but Redis-only worker reads stale data because TTL expired" cell. `DualCache` TTL semantics matter here.
- 3 E2E tests + 1 unit test for a 215-line behavioral feature is on the thin side, but acceptable given the honest disclaimer about async-only / no-per-key-scoping limits, and the load-bearing replay-secret-token test really does prove end-to-end correctness.
- Already MERGED — review value here is post-merge audit / forward-looking nits.

## Verdict: `merge-as-is`

**Rationale:** Already merged. Net assessment: design choices are clean (reuse existing `session_id` + `/spend/logs/session/ui`, no new schema), the load-bearing E2E test (`test_litellm_conversation_id_history_is_replayed_to_model` with planted secret token) actually proves the feature works end-to-end (not just that the data flow plumbing connects), and the limitations are documented honestly upfront ("async-only", "no per-key auth scoping", "multi-worker requires Redis"). Forward nits for follow-up: tighten cross-tenant scoping for both this and `previous_response_id` flows, add `warnings.warn` on sync-path use, plan a `litellm_conv_id_v2_` versioning story before the prefix becomes a public contract.
