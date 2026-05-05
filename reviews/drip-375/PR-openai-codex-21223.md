# openai/codex #21223 — Add workspace headline polling client

- **PR:** openai/codex#21223
- **Head SHA:** `d0bba9e16d125ea880954907cdf792aeb66d5a1f`
- **Files:** 8 changed (+143 / -0) — pure additive, no deletions

## What changed

- `codex-rs/backend-client/src/types.rs:16-50` adds three deserialisable types: `CodexWorkspaceMessagesResponse { messages: Vec<CodexWorkspaceMessage> }`, `CodexWorkspaceMessage { message_id, message_type, message_body, created_at }`, and a `CodexWorkspaceMessageType` enum (`Headline | Announcement | #[serde(other)] Unknown`) with snake_case rename. The `#[serde(other)]` arm is the load-bearing detail — it lets the backend introduce new message types without breaking older clients. Convenience iterators `headlines()` / `announcements()` filter by variant.
- `codex-rs/backend-client/src/client.rs:412-419` adds `Client::list_workspace_messages()` GETting `workspace_messages_url()`. `client.rs:553-560` defines that helper to switch between `/api/codex/workspace-messages` (CodexApi `path_style`) and `/wham/workspace-messages` (ChatGptApi `path_style`) — symmetric with how `list_tasks` already routes. Unit test at `client.rs:884-911` pins both URLs.
- `codex-rs/features/src/lib.rs:225-227, 1116-1121` adds a new `Feature::WorkspaceMessages` flag (key `workspace_messages`, `Stage::UnderDevelopment`, `default_enabled: false`). Gating is honoured at `codex-rs/tui/src/lib.rs:1108-1110` where `prewarm_headline(&initial_config)` only runs when the flag is enabled.
- `codex-rs/tui/src/workspace_messages.rs` (new, 39 lines) implements `prewarm_headline(config)` — spawns a tokio task with a 1000ms `HEADLINE_FETCH_TIMEOUT`, fetches once, and stashes the result in a `static OnceLock<Option<CodexWorkspaceMessage>>`. The `OnceLock` makes it impossible to refetch later in the same TUI process; on timeout/error the slot is set to `None` and stays `None`.

## Risks / notes

- The `OnceLock<Option<...>>` pattern is fire-and-forget by design — there's no refresh path, no retry, and no way for a long-running TUI session to pick up a headline that landed after the 1s timeout. For a "polling client" PR, that's a curious choice; this is closer to "fetch-once headline cache". Worth a follow-up issue to clarify the polling story before this graduates beyond `UnderDevelopment`.
- `workspace_messages.rs` is added but only `prewarm_headline` is wired into `tui/src/lib.rs`. The `announcements()` helper, the `Announcement` variant, and the polling in the PR title are all dead/aspirational at this commit — the PR is genuinely a scaffold. That's fine given the feature flag default is off, but reviewers should not read this as a complete feature.
- `CodexWorkspaceMessage::created_at` is typed as `String` rather than `chrono::DateTime` / RFC3339 — defers parsing to consumers and risks divergence. Minor.
- The `#[serde(other)]` Unknown variant is good forward-compat; the iterator helpers do not surface Unknowns at all, which is the safe default.

## Verdict

**merge-as-is** — feature flag is `default_enabled: false` and `Stage::UnderDevelopment`, so this lands as inert plumbing. The "polling" naming mismatch is worth a follow-up but not a merge blocker since nothing user-visible ships in this PR.
