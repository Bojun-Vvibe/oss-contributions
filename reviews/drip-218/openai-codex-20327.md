---
pr-url: https://github.com/openai/codex/pull/20327
sha: 245b70173128
verdict: merge-as-is
---

# Remove core protocol dependency [4/5]

Penultimate PR in a Sapling-style stack that retires the transitional TUI↔app-server adapter introduced during the hybrid migration. Deletes `codex-rs/tui/src/app/app_server_adapter.rs` (1714 lines) outright plus its companion `codex-rs/tui/src/chatwidget/tests/background_events.rs` (18 lines), with the deletion itself serving as the test: if any TUI flow still depended on the adapter the build would fail. The header comment of the deleted file at `app_server_adapter.rs:1-12` is its own justification — "as more TUI flows move onto the app-server surface directly, this adapter should shrink and eventually disappear" — and this PR is that disappearance. The `#[cfg(test)]` import block at `:65-100` (~35 conditional imports of `codex_protocol::protocol::*` and `codex_protocol::items::*` types like `ExecCommandBeginEvent`, `TurnAbortedEvent`, `TokenUsageInfo`) tells the story: the adapter's whole purpose was bridging two protocol shapes during transition, and once both sides converge on `codex_app_server_protocol` directly the adapter is dead weight.

Verdict is merge-as-is because (a) net diff is `-1732/+0` — a deletion that compiles cleanly is the strongest signal; (b) the PR sits inside a stack where prerequisite consumers were already migrated in #20324/#20325/#20326; (c) Sapling stacks of this shape have a strong "every PR is a single coherent step" discipline that the codex team has been enforcing across drip-211/213.

## what I learned
The right time to delete a transitional adapter is when its `#[cfg(test)]` import block is bigger than its production import block — that's the signal that the adapter has become a test-only fiction propping up a shape that no production caller uses.
