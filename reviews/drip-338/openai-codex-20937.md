# Review: openai/codex #20937

- **Title:** [codex] Thread backend-selected near-limit prompt payload through rate limits
- **Head SHA:** `53dbdbfa9d603270d405d5f8aac78d1844a1e8b0`
- **Scope:** +558 / -3 across 27 files
- **Drip:** drip-338

## What changed

Adds a new `UsageLimitNudge { action, threshold }` payload to the
account-rate-limits update notification surface. `action` is an enum
(`add_credits` | `upgrade`) and `threshold` is a `uint8` percentage. This
lets the backend tell the client *which* nudge UX to display when the user
crosses a usage threshold, instead of the client hard-coding the choice.
Schema files in three flavours (root, v2, per-notification), the protocol
crate, the backend client, the openapi models, and TUI status rendering
are all updated to carry the new field.

## Specific observations

- `codex-rs/protocol/src/protocol.rs:+97` — new `UsageLimitNudge` /
  `UsageLimitNudgeAction` types added with `serde(rename_all =
  "snake_case")` semantics implied by the JSON schema (`add_credits`,
  `upgrade`). Verify the Rust derive matches the schema exactly; the JSON
  uses `snake_case` while many existing protocol enums use `kebab-case`,
  so this is worth a manual eyeball.
- `codex-rs/app-server-protocol/src/protocol/v2.rs:+122` — adds
  `currentUsageLimitNudge: Option<UsageLimitNudge>` on the rate-limit
  update payload. Optional field is the right call: lets older backends
  omit it without breaking deserialization.
- `codex-rs/backend-client/src/client.rs:+66` — backend client now reads
  the new field from the upstream API. Need to verify the upstream key
  name matches `currentUsageLimitNudge` (camelCase) consistently between
  the JSON schema and the openapi-models crate
  (`codex-backend-openapi-models/src/models/usage_limit_nudge.rs:+28`).
- `codex-rs/tui/src/status/tests.rs:+13` and
  `codex-rs/tui/src/chatwidget/tests/status_and_layout.rs:+8` — new TUI
  tests cover the rendering path. Good — a new wire field that doesn't
  exercise the renderer is an easy class of bugs to ship.
- `codex-rs/app-server/tests/suite/v2/rate_limits.rs:+3` — only +3 lines
  of integration coverage on a 558-line protocol surface change feels
  light. Worth one more end-to-end assertion that an `add_credits` nudge
  actually round-trips through `outgoing_message.rs` to a TUI render.
- `codex-rs/app-server-protocol/schema/json/v2/GetAccountRateLimitsResponse.json`
  diff is identical to the notification diff — confirms the field is
  present on both the push notification and the explicit GET, which is
  the right symmetry.

## Risks

- Schema surface change touches both v1 root schema and v2; downstream
  generated clients (TS) must be regenerated in lockstep. The TS schema
  files under `schema/typescript/v2/` are included so this is handled.
- `UsageLimitNudgeAction` is an open enum representation in JSON but a
  closed Rust enum — when the backend adds a third action, deserialization
  will fail hard rather than degrading. Consider `#[serde(other)]` or an
  `Unknown` variant to make this forward-compatible.

## Verdict

**merge-after-nits** — protocol shape and tests look correct; ask for
forward-compat handling on `UsageLimitNudgeAction` and one more E2E
test that exercises the render path through `outgoing_message.rs`.
