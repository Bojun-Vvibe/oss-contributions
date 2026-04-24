# openai/codex#19351 — agents.interrupt_message for interruption markers

**What changed.** New `[agents] interrupt_message` TOML knob (default `true`) resolved into `Config::agent_interrupt_message_enabled`. When `false`, the model-visible "interrupted-turn" history marker is suppressed for both live interrupted turns and interrupted fork snapshots. The client-visible `TurnAborted` event still fires. A new `InterruptedTurnHistoryMarker` enum codifies three states: `Disabled`, `ContextualUser`, `Developer`, selected by config + `MultiAgentV2` feature flag. `core/config.schema.json` regenerated.

**Why it matters.** The interrupted-turn marker is injected into model context so the next turn knows a previous task was deliberately aborted. For some deployments (e.g. ones that drive interruption from an external supervisor and don't want the model to "see" the interruption at all), the marker pollutes context and shows up as confusing developer messages. This gives operators an off-switch without rewriting the abort path.

**Concerns.**
1. **Silent default change risk near MultiAgentV2 toggling.** The new enum's `from_config` returns `Developer` whenever the `MultiAgentV2` feature is enabled and `ContextualUser` otherwise. That coupling is implicit — an unrelated change to feature gating now changes which history role the marker is recorded under. Worth a doc comment on the enum explaining the cross-feature dependency.
2. **Asymmetry between live turn and fork snapshot.** The PR claims both paths honor the flag; reviewers should verify the fork-snapshot path actually plumbs the same enum and not a stale boolean — a partial wiring would produce the worst case (some interruptions visible, some not, depending on whether the abort hit a forked branch).
3. **No round-trip test for the marker round-tripping into the next prompt.** `load_config_resolves_agent_interrupt_message` only checks the bool. The behavioral assertion lives in `disabled_interrupted_fork_snapshot_appends_only_interrupt_event` (per PR description) — make sure the assertion is on the rendered model input, not just the event vector.
4. **Default-true is correct**, but consider documenting that operators who flip to `false` lose the model's ability to reason about why a prior task vanished — that can degrade subsequent turn quality on its own.

Schema regeneration looks clean; additive only.
