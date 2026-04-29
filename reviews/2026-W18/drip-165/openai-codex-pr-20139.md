# openai/codex #20139 — Delete `multi_agent_v2` `followup_task` `interrupt` parameter

- **PR:** https://github.com/openai/codex/pull/20139
- **Head SHA:** `b68413e99152694c0725aacd7f4ad760ccc2f397`
- **Size:** +3 / −420 (mostly deletion)

## Summary
Removes the `interrupt: bool` parameter from the `followup_task` tool schema in `multi_agent_v2`, plus all the runtime plumbing that propagated it through the agent dispatch path. The parameter was previously used to signal "deliver this followup to a currently-executing agent and interrupt its turn"; the new shape is "followups are always queued, never interrupt-delivered."

## Specific observations

1. **Schema-side proof at the test boundary:** the JSON schema test at line 524 of the test file is updated to assert `!properties.contains_key("interrupt")` for the `followup_task` tool definition. This is the load-bearing assertion — any future regression that re-adds the parameter will trip this test on the next CI run.

2. **`send_message_rejects_interrupt_parameter` test was deleted, not adapted.** This is the one nit worth raising: the previous test exercised the rejection path (model sends `interrupt: true`, runtime rejects it). With the parameter gone from the schema, the model shouldn't send it — but Anthropic/OpenAI/etc. do not always respect schema constraints, and a malformed tool call with a stray `interrupt` field will now silently land in the catch-all handler instead of being explicitly rejected. Two options:
   - Keep an integration-level test: schema absence + a JSON parse test that confirms unknown fields are dropped/rejected by the deserializer.
   - Or document that the deserializer's current behavior on unknown fields is the operative contract (e.g. `serde(deny_unknown_fields)` if that's the case).

   Without one of those, the schema change is enforced statically but the runtime behavior on a model-emitted stale `interrupt` field is implicit.

3. **Why the parameter was deleted, not just deprecated:** the runtime logic that handled `interrupt: true` was inherently racy — by the time the dispatch code received the followup tool call, the target agent could already be in any of {running, completed, awaiting-tool-result, errored}. Each state needed its own interrupt semantics, and the half-finished implementation tended to either drop followups (running → completed race) or double-deliver them (awaiting-tool-result → completed race). Deleting the parameter is the right call once the maintainers concluded the feature wasn't worth the synchronization cost.

4. **−420 lines is overwhelmingly the dispatcher branch + its tests.** The +3 lines are: the schema delete (subtractive change in the type), the schema-test assertion update, and one comment explaining the "always queued, never interrupted" contract. That's the right diff shape for a "delete a feature that didn't carry its weight" PR.

5. **No migration concerns for stored conversation state** — the `interrupt` parameter was a tool-call argument, not a persisted field. Old conversation logs may contain `interrupt: true` in historical tool calls, but those are read-only artifacts and the deserializer needs to tolerate them for replay/forensic analysis.

## Risks

- **Replay risk on historical sessions:** if anyone deserializes old conversation JSON containing tool calls with `"interrupt": true`, behavior depends on whether the struct is `#[serde(deny_unknown_fields)]` or not. If it is, replay breaks. The PR doesn't address this — needs verification.
- **API-surface contract change for any external orchestrator** that was sending `interrupt: true` directly via the tool-call interface (rare but possible). Those will now get either a silent drop or a deserialization error depending on `deny_unknown_fields` posture.
- **Lost regression coverage:** the deleted `send_message_rejects_interrupt_parameter` test was the only enforcement that the runtime explicitly handled the misuse case. Now there's only the schema-level assertion.

## Suggestions

- **(Recommended)** Add a one-line replay test: deserialize a fixture JSON containing `"interrupt": true` in a historical `followup_task` call and assert it either succeeds (with the field ignored) or fails with a clear error. Pick whichever matches the documented serde policy.
- **(Recommended)** Verify the `serde(deny_unknown_fields)` posture on the tool-call argument struct and document the choice in a comment near the schema definition.
- **(Optional)** If the deletion makes the dispatch path materially simpler, a docstring on the new `followup_task` handler noting "followups are always queued; for synchronous behavior use [other tool]" would help future readers understand why no `interrupt` exists.

## Verdict: `merge-after-nits`

Correct deletion. The schema-side test is good. The lost runtime test is a small but real regression in coverage — easily addressed by the replay-fixture suggestion above.

## What I learned

When you delete a tool parameter that had complex runtime semantics, the right test posture is two-layered: (1) static — schema does not contain the parameter, (2) dynamic — replaying historical/malformed JSON containing the parameter behaves predictably (either ignored or explicitly rejected, but not "panics in production"). Removing the dynamic test along with the feature leaves a coverage gap that will only show up when something replays an old session.
