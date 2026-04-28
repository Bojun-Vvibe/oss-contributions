# Review — openai/codex #19905

- **Title**: Add compact lifecycle hooks (started by vincentkoc - external contrib)
- **Author**: eternal-openai (Andrei Eternal)
- **Head**: `f050381923c431021be3d6234fa223f631d1dba8`
- **Verdict**: merge-after-nits

## Summary

Adds `PreCompact` and `PostCompact` to the `HookEventName` enum and threads them across hook config, discovery, dispatch, app-server protocol notifications, generated JSON+TS schemas, analytics event-name mapping, and TUI hook-event rendering. Wires `PreCompact` *before* both local and remote compaction with `manual`/`auto` trigger matching plus `custom_instructions` in the input; `PreCompact` can block via exit code 2 stderr or JSON `{"decision":"block","reason":"..."}`; `PostCompact` fires after success with `compact_summary` in the input. Originated from the external #19060 contribution.

## Findings

- **Right shape for a lifecycle gate**: `PreCompact` runs *before* context is rewritten and can block, which is the only useful semantics for a policy hook — blocking after the fact would just be a logging surface. The `manual`/`auto` trigger discriminator on the input lets policies independently match user-initiated `/compact` vs. automatic context-pressure compaction, which are very different security/audit surfaces.
- **Exhaustive-match propagation works correctly**: the `v2_enum_from_core!` macro at `protocol/v2.rs:436-440` adding both variants in one place forces every downstream `match HookEventName` to be updated or stop compiling. Verified the analytics mapping at `analytics/src/events.rs:671-677` and the schema regeneration both got both variants — no half-add risk.
- **`ManagedHooksRequirements` schema gets two new required fields** (`PreCompact`, `PostCompact`) at `v2.rs:972-980` and the JSON schema's `required` array at `codex_app_server_protocol.schemas.json:10199-10215` adds both. **This is a breaking change for external strict-schema v2 consumers** parsing `ManagedHooksRequirements` with `additionalProperties: false`-style validators, but PR title/body don't carry a `BREAKING:` marker. Either annotate as breaking or land alongside a deprecation note in the v2 protocol changelog.
- **`compact_summary` in `PostCompact` input** is the right shape — a summary-aware audit hook needs the actual collapsed text to log/redact, not just "compaction happened". Make sure the documented hook contract names this field with its expected serialization (Markdown? plain text?) so script authors don't guess.
- **JSON-block syntax `{"decision":"block","reason":"..."}` for `PreCompact`** matches the pattern already used by other blocking hooks — good consistency. One nit: the PR body doesn't show the wire shape for `decision` values other than `"block"` (e.g. is `"allow"` accepted and meaningful, or is omission the only allow-path?). Document explicitly to prevent script authors over-specifying.
- **Integration tests cited** (`compact_hooks_respect_matchers_and_post_receives_summary`, `manual_pre_compact_hook_blocks_compaction`) at `core/tests/suite/compact.rs` cover the matcher-filtering and blocking-behavior paths — both are the right cells to pin since a future "let's simplify by always running on both manual and auto" change would silently violate the documented manual/auto independence.
- **Analytics event names are PascalCase strings** (`"PreCompact"`, `"PostCompact"`) at `analytics/src/events.rs:674-675` — matches the `HookEventName` Rust variant casing but contrasts with the `"preCompact"`/`"postCompact"` JSON serialization. The split is intentional (analytics keys vs. wire format) but a one-line code comment would help a future reader who notices the inconsistency.

## Recommendation

Merge after (a) adding a `BREAKING:` or `protocol-v2 minor bump` annotation reflecting the two new required fields in `ManagedHooksRequirements`, (b) documenting the wire contract for `compact_summary` and any non-`"block"` `decision` values, and (c) a one-line comment on the PascalCase-vs-camelCase analytics-name split. The architectural shape is right and the test coverage is the right cells.