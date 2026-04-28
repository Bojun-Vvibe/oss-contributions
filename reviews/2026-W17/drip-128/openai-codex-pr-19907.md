# openai/codex #19907 — Clarify network approval auto-review prompts

- URL: https://github.com/openai/codex/pull/19907
- Head SHA: `03fc0b7e0177aae5828c181283d89272db740cc9`
- Size: +93 / -8 across 2 files
- Verdict: **merge-as-is**

## What the change does

When the guardian asks for approval on a network access request, the prior
prompt template treated network-access exactly like any other action: it
emitted "Retry reason:" + the generic blocked-by-policy string, then "Planned
action JSON:" with the network-access payload. That framing pushed the
auto-review LLM to focus on whether the *network connection* was authorized,
which is the wrong question — the user authorizes commands, and a network
connection is a downstream consequence of the triggering command (e.g. `curl
example.com` → connection to example.com:443).

Fix: split the prompt rendering at
`codex-rs/core/src/guardian/prompt.rs:177-211` on the
`GuardianApprovalRequest` variant. The `NetworkAccess` arm at `:178-202`
emits a dedicated "Network approval context:" block that:

1. Names the target explicitly: `"This approval request is about network
   access to the target in the network access JSON below
   (`{target}`)."`
2. If a `trigger` is present, redirects the assessor to the triggering
   command: `"focus primarily on whether the triggering command is authorised
   by the user and whether it is within the rules. The user does not need to
   have explicitly authorised this exact network connection, as long as the
   network access is a reasonable consequence of the triggering command."`
3. If `trigger` is `None`, falls back to: `"No trigger action was captured
   for this network access request. When performing the assessment, use the
   retained transcript and network access JSON to evaluate user authorization
   and risk."` — the right backstop, doesn't pretend a missing trigger means
   "approve by default".
4. Closes with `"Network access JSON:"` instead of `"Planned action JSON:"`.

The `_` arm at `:204-210` keeps the legacy "Retry reason: ... Planned action
JSON:" rendering for every other approval request type — non-NetworkAccess
behaviour is byte-identical to before.

## What is load-bearing

The `retry_reason` is intentionally **dropped** from the network-access
branch — that's the whole point of the change. Prior code surfaced strings
like `"Network access to \"example.com\" is blocked by policy."` as the
"Retry reason:", which is misleading because that's a description of the
block, not a reason the assessor should care about. The test at
`codex-rs/core/src/guardian/tests.rs:850-855` pins this with
`assert!(!text.contains("Planned action JSON:"))`,
`assert!(!text.contains("Retry reason:"))`, and
`assert!(!text.contains("Network access to \"example.com\" is blocked by
policy."))` — that last assertion is the regression anchor for
"don't reintroduce the misleading retry reason in this branch."

The test also pins the positive shape: the four key sentences must appear
verbatim (`:830-849`), and the trigger payload (`"trigger"` JSON key) and the
new section header (`"Network access JSON:"`) must appear. Together those
six asserts form a complete spec for the prompt template — any future
wording tweak has to either preserve them or update the test deliberately.

## Risk

- The trigger-missing branch still emits "use the retained transcript and
  network access JSON to evaluate user authorization and risk" which leans on
  the assessor having parent-history context. The test at `:803`
  (`seed_guardian_parent_history`) is the trigger-present case — there is no
  symmetric test for the trigger-missing branch. Not a blocker because the
  code path is small and the prose is correct, but a follow-up test for
  `trigger: None` with `seed_guardian_parent_history` would close the gap.
- No new strings added that need translation/i18n — same English-only template
  as the rest of `prompt.rs`.

## Recommendation

Ship. Right diagnosis (network-access is a consequence-of-command, not a
first-class authorization decision), right prose, right placement (variant
match keeps the legacy branch byte-identical), right test (positive +
negative asserts as a spec).
