# openai/codex #19896 — Update models.json (gpt-5.4 default reasoning level)

- PR: https://github.com/openai/codex/pull/19896
- Author: app/github-actions (bot — automated update)
- Head SHA: `39194aabbbee`
- Base: `main` · State: OPEN
- Diff: 1+/1- across 1 file (`codex-rs/models-manager/models.json`)

## Verdict: needs-discussion

## Rationale

- **Single-field default-flip with non-trivial UX impact.** At `models.json:114`, `gpt-5.4`'s `default_reasoning_level` changes from `"xhigh"` to `"medium"`. This is a behavioral default that ships to every user who selects `gpt-5.4` without explicitly overriding `reasoning_level`. The supported levels list at `:115-` is unchanged (low/medium/high/xhigh all still selectable), so the upper bound of capability is preserved — but the *out-of-the-box latency, cost, and answer-quality* characteristics of the model from the user's perspective shift materially. `xhigh → medium` typically means lower latency, lower per-call cost, and shorter chain-of-thought before the model commits to an answer.
- **No PR body, no rationale, no test plan.** The body is literally "Automated update of models.json." That's appropriate for a *mechanical* sync (e.g., adding a new model, bumping a price, updating a context window from upstream metadata) but inappropriate for a default-behavior flip. There's no link to the upstream change that triggered this, no benchmark/eval data showing `medium` is now the right default for `gpt-5.4`, no comparison of regression risk across the existing user base, and no rollout/canary plan for what is effectively a global UX change.
- **Bot-authored default-behavior changes are a structural risk.** The author signature `app/github-actions` and `is_bot: true` means a humans-in-the-loop review is the *only* gate on a default-flip that affects every user of the listed model. The diff shape (`-`/`+` on a single field) makes the change easy to miss in skim-review. This is the canonical class of "low-LOC PR with high blast-radius" that benefits most from explicit reviewer attention.
- **The change *might* be correct.** `gpt-5.4` may have been recently re-tuned upstream such that `medium` reasoning is now the cost/quality sweet spot, or the prior `xhigh` default may have been an over-specification carried from `gpt-5.3`. Both are plausible. But the PR body provides zero evidence either way — the reviewer is being asked to ratify a behavioral default-flip with no decision-input.

## What needs discussion

1. **What upstream change triggered this?** Is there a model-card revision, a price change, or an internal eval that supports `medium` as the new default? Link or note in the PR body.
2. **What's the user-visible impact?** A 2-row before/after table showing typical token counts and wall-clock latency at `xhigh` vs. `medium` for one canonical task (`fix this build error`, `add a feature`, etc.) would let reviewers calibrate.
3. **Should the changelog/release notes call this out?** Default-flips on a model that users have already integrated into their config files deserve at least a one-line note in the next release. Users who tuned their cost/latency budget around the prior `xhigh` default will see a step-change without warning.
4. **Is the same revision needed for sibling models?** If `gpt-5.4` is moving from `xhigh` → `medium`, are `gpt-5.3` / `gpt-5.5` / `gpt-5.5-codex` (if present) on consistent defaults, or does this introduce inter-model inconsistency that will surprise users switching models?

## What I learned

Bot-authored "Update models.json" PRs are the right shape for *mechanical* metadata sync but the wrong shape for *default-behavior* changes. The structural fix is to split the bot's job into two PR templates: (a) "metadata sync" — context-window, pricing, supported_levels list, alias additions — auto-mergeable with one human reviewer; and (b) "default-behavior change" — `default_reasoning_level`, `default_temperature`, `default_max_tokens` — never auto-mergeable, requires PR body with eval link + before/after numbers + release-note hook. The cost of conflating them is exactly this situation: a one-line diff that's structurally a UX change but reads like a metadata bump, and the only defense is an attentive human reviewer who happens to notice the field name.
