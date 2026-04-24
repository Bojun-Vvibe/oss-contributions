# openai/codex#19360 — surface multi-agent thread limit in spawn description

**What changed.** Adds a `max_concurrent_threads_per_session` field to `ToolsConfig` and `SpawnAgentToolOptions`, threads it from `Config::agent_max_threads` through `TurnContext`, the review thread spawner, and per-turn session contexts, and renders a sentence into the `spawn_agent` tool description (`This session is configured with max_concurrent_threads_per_session = N for concurrently open agent threads.`). Tests updated in `agent_tool_tests.rs` and `spec_tests.rs`.

**Why it matters.** The model previously had no in-band signal of the host's concurrency cap, so a sub-agent dispatcher would happily try to fan out N+1 spawns and only learn about the limit from a runtime rejection. Putting the number in the tool description converts a runtime failure into prompt-time planning, which is the right shape for a multi-agent control surface.

**Concerns.**
1. **Silent omission when unset.** `concurrency_guidance` falls back to an empty string when `agent_max_threads` is `None`. The model then sees no mention of any limit — distinguishable only from a config in which "unlimited" is meaningful. Worth either always emitting the line (with `unlimited`) or documenting that absence implies unbounded.
2. **Single-source-of-truth drift.** `Config::agent_max_threads` is the same value advertised in PR #19354 (`max_concurrent_threads_per_session` alias). If the alias resolution lands after this PR, this rendered text is the externally-visible name — verify the spelling matches what users actually set in TOML, not just the internal field name.
3. **Wording is brittle.** A single literal sentence is now asserted by exact regex in two test cases (`spawn_agent_description_omits_usage_hint_when_disabled`, `..._uses_configured_usage_hint_text`). Any future copy edit will need a triple update. Worth extracting the template.
4. **No assertion that the value matches `agent_max_threads`** — tests hard-code `Some(4)`. A regression that wires the wrong field (e.g. `agent_max_depth`) would pass.

Otherwise straightforward; the additive `with_*` builder fits existing style.
