# Review: All-Hands-AI/OpenHands#14122 — Enable sub-agent delegation via TaskToolSet

- **PR**: https://github.com/All-Hands-AI/OpenHands/pull/14122
- **State**: OPEN
- **Author**: xingyaoww (Xingyao Wang)
- **Range**: +4 / −0
- **Head SHA**: `90b5993c3173db3c8fbb6c94991b770cf3417132`
- **Base SHA**: `137bede1f5e867a7eb17ae36e78a05ffef6ec09a`
- **Verdict**: needs-discussion
- **Reviewer date**: 2026-04-25

## What the PR does

In `openhands/app_server/app_conversation/live_status_app_conversation_service.py:1305–1310`,
when `_build_start_conversation_request_for_user` is in the
"default tools" branch (i.e. not the planning-tools branch),
appends `Tool(name=TaskToolSet.name)` if
`user.agent_settings.enable_sub_agents` is True. Adds two
imports: `Tool` from `openhands.sdk.tool.spec` and `TaskToolSet`
from `openhands.tools.task`.

## Observations

1. **Correct minimal wiring, but the change has a hidden
   dependency.** PR body says "Depends on: SDK PR #2948 (adds
   `enable_sub_agents` boolean to `LLMAgentSettings`)". That
   SDK PR must merge and ship in a release **before** this PR's
   `user.agent_settings.enable_sub_agents` access stops being a
   `KeyError` / `AttributeError` for any user who hasn't migrated
   their settings. The diff includes no defensive
   `getattr(user.agent_settings, "enable_sub_agents", False)` — it
   trusts the SDK schema. If SDK pin in
   `pyproject.toml` / `requirements*.txt` isn't bumped in the
   same merge, `live_status_app_conversation_service` crashes
   on every conversation start.
2. **The gate is per-user, default-off — good rollout
   discipline.** PR body emphasizes this; the code matches it
   exactly. Not in this PR's scope, but the Settings UI to
   toggle the bool needs to ship for the feature to be usable
   end-to-end.
3. **No metric / log on enablement.** When a user enables
   sub-agents, nothing on the server side records the toggle,
   which makes adoption analytics impossible. PR body notes
   "task tool description adds ~800+ tokens to the system
   prompt" — that's a real operator cost. Worth a single
   `logger.info("sub_agents enabled for user=%s", user.id)`
   on the append branch, or a counter increment if
   prometheus is wired.
4. **Risk: cost amplification.** Every sub-agent delegation
   spawns a `LocalConversation` "in the same process,
   consuming additional LLM calls" (per PR body). For users on
   a fixed billing plan, a single enable-and-forget toggle
   could 3–10x their LLM spend depending on how aggressively
   the main agent delegates. This deserves either (a) a
   second confirmation in the Settings UI, or (b) a per-user
   sub-agent budget cap. Out of scope for this PR but worth
   tracking.
5. **`TaskToolSet.name` should be the only contract surface.**
   The diff uses `Tool(name=TaskToolSet.name)` rather than
   passing the whole `TaskToolSet` instance. That's correct —
   the tool router resolves by name from the registry. Means
   future SDK upgrades can change the toolset internals
   without breaking this wiring.
6. **No test in this PR.** The change is small enough that a
   single test covering "enable_sub_agents=True → tools
   contains TaskToolSet name" and "enable_sub_agents=False →
   does not" would close the loop and ensure the SDK schema
   stays compatible. The omission is the main reason I'm not
   `merge-as-is`.

## Verdict reasoning

Tiny diff, real feature, but it reads as the *deployment*
slice of a feature whose configuration plumbing (SDK PR
#2948) and observability (Settings UI, logging, billing
guardrails) live elsewhere. I'd want (a) confirmation the SDK
dependency is pinned and shipped, (b) a defensive
`getattr(..., False)`, and (c) a one-line happy/sad-path
test, before this lands. Hence `needs-discussion`.

## What I learned

Feature-flag-by-user-setting is good rollout policy, but
the enable side and the cost-control side need to ship
together. A boolean toggle that can 5x someone's LLM bill
without warning is a UX trap, even when the feature is
opt-in.
