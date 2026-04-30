---
pr-url: https://github.com/openai/codex/pull/20502
sha: 6777a85b8252
verdict: merge-after-nits
---

# fix(tui): set persist_extended_history: false

Four-site policy flip from `persist_extended_history: true` to `false` across the TUI's thread-lifecycle params: `app_server_session.rs:1178` (start), `:1209` (resume), `:1243` (fork), and the corresponding initial-history fork path in `codex_message_processor.rs:2595`. No control-flow change, no test churn — pure default-value adjustment. The change rides on top of the `EventMsg::McpToolCallEnd`/`ApplyPatchEnd` policy moves in #20464/#20463 which made *limited* history mode lossless-enough for normal session replay, so flipping the TUI default off the extended path is now safe.

The "extended" history mode persists every streaming `outputDelta` and per-event metadata, intended for offline analytics and crash forensics, at the cost of multi-MB rollout files for routine sessions. The TUI is the wrong consumer for that data — it's the *user* of the rollout, not an analytics pipeline — so the previous `true` was effectively forcing every interactive user to pay the disk-I/O and replay-cost tax for a feature only the offline tooling wanted. Inverting the default at the four call sites is the right shape: it leaves the field on `ThreadStartParams`/`ThreadResumeParams`/`ThreadForkParams` for explicit opt-in by analytics callers, but stops the TUI from being the involuntary opt-in.

Nits: (1) the four sites all set the same value but live in three different functions in `app_server_session.rs` plus one in `codex_message_processor.rs` — a `const TUI_PERSIST_EXTENDED_HISTORY: bool = false` would prevent future drift where one site flips back to `true` for a debug session and never gets reverted; (2) no test pins the contract that "TUI sessions write rollout files of bounded size in steady state" — the regression that prompted this PR (presumably oversized rollouts) has no test surface to prevent recurrence; (3) the commit message / PR description should call out that this is a behavior change for users who were intentionally relying on extended-mode rollout files for their own tooling — there's a non-zero migration audience.

## what I learned
A boolean default that's set in 4 different params structs across 2 files is a refactor-debt smell — when the right value flips, the diff is "trivially correct" but visually trivially-easy-to-miss-one-site. Lift the value into a named constant, even if the params shape doesn't have a natural slot for it, so the next flip is one line not four.
