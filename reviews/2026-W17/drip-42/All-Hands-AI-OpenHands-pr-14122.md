# All-Hands-AI/OpenHands PR #14122 — feat: enable sub-agent delegation via TaskToolSet in app server

- **URL:** https://github.com/All-Hands-AI/OpenHands/pull/14122
- **Head SHA:** `90b5993c3173db3c8fbb6c94991b770cf3417132`
- **Files touched:** 1 (`openhands/app_server/app_conversation/live_status_app_conversation_service.py`)
- **Verdict:** `request-changes`

## Summary

Wires the SDK's already-implemented `TaskToolSet` (sub-agent
delegation) into the app server's default tool list, gated by a new
per-user `enable_sub_agents` boolean on `LLMAgentSettings`. When the
user toggles it on in Settings, the main agent gets the `task` tool
and can delegate to built-in sub-agents (`bash-runner`,
`code-explorer`, `general-purpose`, `web-researcher`).

## Specific references

- `openhands/app_server/app_conversation/live_status_app_conversation_service.py:1305-1311`
  — inside `_build_start_conversation_request_for_user`, after
  `tools = get_default_tools(enable_browser=True)` the diff appends
  `Tool(name=TaskToolSet.name)` only when
  `user.agent_settings.enable_sub_agents` is `True`.
- Two new imports at lines ~95 and ~108: `from openhands.sdk.tool.spec
  import Tool` and `from openhands.tools.task import TaskToolSet`.

## Reasoning

The change itself is the right shape — minimal, gated, default-off,
opt-in via Settings UI — exactly the rollout posture a non-trivial
agent capability should land with. No existing flow regresses when
`enable_sub_agents` is `False`.

However, this is `request-changes` because of a hard dependency the
PR explicitly calls out: it depends on
**software-agent-sdk PR #2948** which adds the
`enable_sub_agents` field to `LLMAgentSettings`. Merging this PR
first against any SDK that does not yet expose the field will raise
`AttributeError` on every conversation-creation call — a hard 500 in
the hot path, not a recoverable degradation.

Required before merge:
1. SDK PR #2948 must be merged AND the `openhands-tools` /
   `openhands-sdk` pinned version in this repo's `pyproject.toml`
   must be bumped to a version that contains it. The current PR diff
   does not touch dependency pins.
2. Defensive `getattr(user.agent_settings, 'enable_sub_agents',
   False)` would be a small belt-and-braces guard against partial
   upgrades / older settings rows in the DB.
3. Add at least one app-server-level test that asserts (a) the tool
   is absent when the flag is off and (b) present when it's on. The
   PR ships zero tests against the new branch.

Logically sound, sequencing is the issue.
