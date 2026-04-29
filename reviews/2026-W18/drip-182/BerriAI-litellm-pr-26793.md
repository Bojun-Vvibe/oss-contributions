---
pr: BerriAI/litellm#26793
sha: 9d9efc1afba6fd1d6421f775a9a150524ad27a5c
verdict: needs-discussion
reviewed_at: 2026-04-29T18:31:00Z
---

# feat(proxy): durable agent workflow run tracking via /v1/workflows/runs

## Context

Adds three new Prisma models (`LiteLLM_WorkflowRun`,
`LiteLLM_WorkflowEvent`, `LiteLLM_WorkflowMessage`) and 8 REST endpoints
under `/v1/workflows/runs` in
`litellm/proxy/management_endpoints/workflow_management_endpoints.py`.
The motivating use case: long-running agents (the PR mentions
"shin-builder") whose conversation state and step history were
in-memory only, so a process restart wiped everything.

## What's good

- Generic schema design — `workflow_type`, `step_name`, and `data` are
  caller-defined strings/JSON. No hardcoded stage names baked into
  the proxy. The status auto-update map (`_EVENT_STATUS_MAP` at the top
  of the file: `step.started→running`, `hook.waiting→paused`, etc.)
  is small and overridable via `PATCH /v1/workflows/runs/{run_id}`.
- The `session_id` bridge to `LiteLLM_SpendLogs.session_id` is a clean
  reuse: existing `/ui/spend_logs/view_session_spend_logs?session_id=`
  works for free. No new cost-tracking machinery needed.
- Comma-separated status filter (`?status=running,paused`) handled
  cleanly in `list_workflow_runs`: splits and emits `{"in": statuses}`
  only when there's more than one. That's the correct Prisma idiom.

## Concerns

1. **Sequence number race.** `_get_next_sequence_number` does
   `MAX(sequence_number) + 1` then the caller does an insert. Two
   concurrent appenders racing on the same `run_id` will both compute
   the same next value and one will conflict on the unique index (or
   silently overwrite, depending on schema constraints not shown in
   the diff). For an agent that may have multiple step emitters or
   hook receivers, this is a real failure mode. Either:
   (a) wrap the read+insert in a Prisma transaction with
       `SERIALIZABLE` isolation, or
   (b) use a Postgres sequence / `RETURNING sequence_number` with a
       partitioned-by-run sequence.
2. **No authorization scoping.** All endpoints depend on
   `user_api_key_auth` but don't filter `where` by the calling key's
   tenant/team. Any authenticated key can `GET /v1/workflows/runs/{any_id}`
   and read another team's conversation. The conversation-message table
   stores full content (the PR explicitly notes spend logs truncate at
   `MAX_STRING_LENGTH_PROMPT_IN_DB` while this doesn't), so the blast
   radius of a missing scope check is high.
3. **`raise HTTPException(status_code=500, detail=str(e))`** leaks
   raw Prisma error messages (table names, column names, sometimes
   query fragments) to the caller. Standard advice: log with
   `verbose_proxy_logger.exception(...)` (already done) and return
   a generic 500 message.

## Verdict

`needs-discussion` — the data model and endpoint shape are good, but
the concurrency model and tenancy scoping need to be settled before
this goes live. The error-leak issue is a small follow-up.
