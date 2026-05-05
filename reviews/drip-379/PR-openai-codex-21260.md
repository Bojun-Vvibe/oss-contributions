# openai/codex #21260 — [codex] Move thread naming to app server

- URL: https://github.com/openai/codex/pull/21260
- Head SHA: `96bcac6704dc815ff834c12e3b34b189f771fb2b`
- Author: pakrym-oai
- Size: +53 / -223 across 13 files

## What it does

Migrates thread-naming from a core `Op::SetThreadName` event-driven path to a direct app-server-side `update_thread_metadata(ThreadMetadataPatch { name: Some(name), .. }, /*include_archived*/ false)` call. Deletes the `EventMsg::ThreadNameUpdated` rollout-line emission path entirely (no longer persists name-change events into rollout files), removes the `ThreadNameUpdatedNotification` translation in `bespoke_event_handling.rs:1209-1217`, and consolidates the loaded-thread vs not-loaded-thread branches in `thread_processor.rs:1346-1366` so both paths now go through the same `StoreThreadMetadataPatch` API. Drops the rollout-count assertion from the websocket integration tests (the rollout file no longer contains a `ThreadNameUpdated` line by design).

## Observations

- The two branches at `thread_processor.rs:1346-1366` (loaded thread → `thread.update_thread_metadata`; not loaded → `self.thread_store.update_thread_metadata`) duplicate the `StoreThreadMetadataPatch { name: Some(name.clone()), ..Default::default() }` literal. Could be hoisted to a single `let patch = ...` above the `if let Ok(thread)` to make the intent (same patch, two persistence paths) explicit. Not a blocker.
- `include_archived: false` is the chosen safety default, which matches the previous behavior — renaming an archived thread still requires explicit unarchive. Worth a one-line code comment because the magic-bool is easy to misread.
- **Behavior change worth flagging in release notes:** rollout files no longer contain `ThreadNameUpdated` lines. Anything downstream that replays rollouts to reconstruct historical names (debug tooling, support-export pipelines, third-party rollout consumers) will lose that signal. The PR title says "move to app server" but the user-visible contract change is "stop persisting in rollout."
- The deleted `thread_name_update_rollout_count` test helper in `thread_name_websocket.rs:172-194` had been pinning that exact contract — its removal is correct given the new design but underscores that the rollout-line emission was a tested guarantee until now.
- `external_agent_config_processor.rs:322-328` correctly switches from `submit(Op::SetThreadName)` to `update_thread_metadata(ThreadMetadataPatch { name: Some(name), .. }, false)` for the imported-session naming path, keeping the import flow consistent with the main rename flow.

## Verdict: needs-discussion

The plumbing change is clean and the diff is a net -170 lines, which suggests the right direction. But the silent drop of `ThreadNameUpdated` rollout-line persistence is a contract change that affects rollout consumers, and the PR description doesn't enumerate which downstream consumers were audited. Worth confirming with maintainers whether (a) any internal/external tooling depends on those rollout lines, and (b) if a brief CHANGELOG / migration note is warranted before merge.