# openai/codex #20149 — Move request input mode gating into tools

- **PR:** https://github.com/openai/codex/pull/20149
- **Head SHA:** `7988650c0ecb1c916ce9825f30ac7dd1bb240408`
- **Size:** ~+250 / −400 (net −150) across app-server, core/tools, models-manager, prompt_debug

## Summary
Removes the `default_mode_request_user_input` feature-gated branching that lived inside the collaboration-mode preset templates and the `RequestUserInputHandler`, and replaces it with a single source of truth: the tool registry. The handler now carries a `Vec<ModeKind> available_modes` and decides at invocation time based on `session.collaboration_mode().mode` membership. Preset prompts are simplified — they no longer template-interpolate `{{REQUEST_USER_INPUT_AVAILABILITY}}` / `{{ASKING_QUESTIONS_GUIDANCE}}` per feature flag; instead they instruct the model to "Use the `request_user_input` tool only when it is listed in the available tools for this turn." Cleaner: behavior is inspected at the place that owns it (the tool spec), not duplicated into the prompt template.

## Specific observations

1. **The load-bearing handler change at `core/src/tools/handlers/request_user_input.rs:14-16`:**
   ```rust
   pub struct RequestUserInputHandler {
   -    pub default_mode_request_user_input: bool,
   +    pub available_modes: Vec<ModeKind>,
   }
   ```
   Plus the call-site at `:46-49` becomes `request_user_input_unavailable_message(mode, &self.available_modes)`. This is the right shape — the gate is now expressed as data ("which modes is this tool valid in") rather than as a feature-flag branch ("Default mode is special-cased").

2. **Spec wiring at `core/src/tools/spec.rs:166-172`:**
   ```rust
   let request_user_input_handler = Arc::new(RequestUserInputHandler {
   -    default_mode_request_user_input: config.default_mode_request_user_input,
   +    available_modes: config.request_user_input_available_modes.clone(),
   });
   ```
   The config field rename is the real protocol change — `default_mode_request_user_input: bool` → `request_user_input_available_modes: Vec<ModeKind>`. Anyone with a saved config that used the boolean key will silently get the field's `Default` (probably empty Vec, which means "never available"). A migration shim or one-release-cycle deprecation read would prevent silent breakage.

3. **Removal of `CollaborationModesConfig` plumbing** from at least 8 sites (app-server `codex_message_processor.rs:851`, `:6457`, `models-manager/src/manager.rs:180`, `prompt_debug.rs`, multiple test files). Net reduction in coupling — `CollaborationModesConfig` was a one-field struct that existed only to thread the `default_mode_request_user_input` flag through preset rendering. With the gate moved to the tool, the struct is no longer needed.

4. **The two helper functions `request_user_input_availability_message` and `asking_questions_guidance_message`** in `models-manager/src/collaboration_mode_presets.rs` (~lines 53-90 in the deleted block) are gone. Their purpose was to inject conditional copy into the preset based on the flag. The replacement copy at `core/src/collaboration-mode-templates/templates/default.md:170-173` is unconditional:
   > Use the `request_user_input` tool only when it is listed in the available tools for this turn.

   This is conceptually cleaner — the model is told to inspect the tool list, not to hard-code a mode-specific assumption. The risk is that models that don't check the tool list reliably (or don't get the tool list at all on resume) will either over- or under-call `request_user_input`. The handler-side gate at `request_user_input.rs:46-49` is the safety net.

5. **Test rename at `models-manager/src/collaboration_mode_presets_tests.rs`** drops `default_mode_instructions_use_plain_text_questions_when_feature_disabled` and folds the assertion `assert!(default_instructions.contains("ask the user directly with a concise plain-text question"))` into the consolidated `default_mode_instructions_replace_mode_names_placeholder` test. The "feature-disabled" branch is now untestable because the feature flag is gone — that's the *point* of this refactor — but it removes one assertion that the prompt copy is always plain-text-fallback friendly. Not a regression, but worth keeping as a non-feature-gated assertion.

6. **README change at `app-server/README.md:194-194`** adds the line "Built-in presets do not select a model or reasoning effort." which is a separate behavioral commitment — and the test-side change `assert_eq!(plan_preset().model, None); assert_eq!(plan_preset().reasoning_effort, None);` (replacing the `Some(Some(ReasoningEffort::Medium))` assertion) confirms the preset now leaves both model and reasoning effort unset. This is a meaningful behavioral shift bundled into a "tool gating refactor" PR — clients that relied on the preset selecting Medium reasoning effort for Plan mode will see different defaults. Worth either splitting out or release-noting prominently.

## Risks

- **Config migration:** the `default_mode_request_user_input: bool` → `request_user_input_available_modes: Vec<ModeKind>` field rename has no documented compat shim. Users with that key in their `config.toml` will silently lose the configured behavior. Either a `#[serde(alias = "default_mode_request_user_input")]` with a one-release-cycle deprecation warning, or a startup config-validation pass that flags the dead key.
- **Behavioral coupling between the README's new "presets do not select a model or reasoning effort" promise and the test that asserts it** — these two changes belong together but are bundled with the unrelated tool-gating refactor. A rollback of the tool gating would also un-document the model/reasoning behavior, which may be undesirable.
- **Removal of `default_mode_instructions_use_plain_text_questions_when_feature_disabled`** drops the non-feature-gated assertion that the prompt copy contains "ask the user directly with a concise plain-text question." Worth keeping that assertion in the surviving test — it pins prompt-copy regression even after the feature flag goes away.
- **Models that don't honor "only call tools listed in this turn"** — most do, some don't (looking at older Claude / open-source LLMs). The handler-side gate at `request_user_input.rs:46-49` catches these; the user-visible symptom would be a turn where the model emits `request_user_input`, gets `RespondToModel(unavailable_message)`, and resubmits. Worth explicit telemetry on this rejection rate to size the impact.

## Suggestions

- **(Recommended)** Add `#[serde(alias = "default_mode_request_user_input")]` to the new `request_user_input_available_modes` field (or a startup config-load warning) so users with the old key get a migration path, not silent breakage.
- **(Recommended)** Split the "presets don't select model/reasoning" change out of this PR or call it out in the title / release notes — it's a separate behavioral commitment that's currently buried under "tool gating refactor."
- **(Recommended)** Re-add a non-feature-gated assertion in the surviving preset test that pins the "ask the user directly with a concise plain-text question" copy. Future prompt-copy edits shouldn't silently drop it.
- **(Optional)** Add a `tracing::debug!` or counter in `RequestUserInputHandler::handle` when `available_modes` rejects an invocation, so the rate of model-mistake `request_user_input` calls is observable.

## Verdict: `merge-after-nits`

Correct refactor — the gate now lives in the tool spec, which is the right architectural layer. The diff is net-negative LOC and removes a templating coupling that was always going to drift. The nits are config migration shim, scope (split out or release-note the preset model/reasoning change), and one regression-test assertion to re-add.

## What I learned

When a feature flag changes "what tools the model sees" *and* "what the prompt template renders," the prompt-template branch is almost always the wrong layer for the gate — because the model can ignore prompt instructions but cannot ignore a missing tool. Moving the gate to the tool spec collapses two divergent codepaths into one and lets the prompt copy be unconditionally aspirational ("use this tool when listed"). This PR is the canonical shape for that refactor.
