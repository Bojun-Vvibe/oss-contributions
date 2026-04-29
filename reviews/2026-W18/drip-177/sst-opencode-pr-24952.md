# sst/opencode#24952 — feat(agent): enhance agent capabilities with review/test agents and self-reflection

- **Repo:** sst/opencode
- **PR:** [#24952](https://github.com/sst/opencode/pull/24952)
- **Head SHA:** `c10bf75209d260adc86f1f991f99ea6a951ee57d`
- **Author:** xjwm5685-ui
- **Size:** +515 / -287 across 33 files

## Summary

Adds two new built-in subagents (`review`, `test`), wires an experimental
`self_reflect` cfg flag onto the `build` agent's prompt, and rewrites the
`explore` agent prompt from prose into a numbered/structured form. PR body
also claims "20+ security/reliability fixes" but the diff is mostly prompt
text plus an agent-registration block — there's not 20 fixes in here.

## What's actually going on

Three logically-separate changes packaged as one PR:

1. **Two new subagent registrations** at `packages/opencode/src/agent/agent.ts:242-289`:
   - `review` — `mode: "subagent"`, permissions deny-all-then-allow read-only
     tools (`grep`, `glob`, `list`, `read`, `bash`) plus `external_directory`
     allow-list. Prompt at `packages/opencode/src/agent/prompt/review.txt` (27
     lines, severity tiers + checklist + output format).
   - `test` — `mode: "subagent"`, permissions allow `edit`/`write`/`read`/
     `bash`/`grep`/`glob` (write-capable, unlike review). Prompt at
     `packages/opencode/src/agent/prompt/test.txt` (30 lines).
2. **`build` agent prompt becomes conditional** at `agent.ts:109-114`:
   ```ts
   const selfReflectPrompt = cfg.experimental?.self_reflect
     ? `\n# Self-Reflection\nBefore completing:\n1. ...`
     : undefined
   const buildAgentPrompt = `# Verification\nBefore completing: requirements met, ...`
   ```
   then `prompt: selfReflectPrompt ?? buildAgentPrompt` at `agent.ts:130`.
3. **`explore` prompt rewrite** at `prompt/explore.txt` — converts a 14-line
   prose description into a 4-section numbered playbook. Substantively the
   same instructions, mostly a stylistic delta.

Two glaring issues stand out in the diff:

**(a) Duplicate `options` key in `.oxlintrc.json` is now removed.** Lines 44-49
of the old config had two consecutive `"options": { "typeAware": true }` blocks
— the diff drops *both*, not just the duplicate. That means `typeAware: true`
is *gone*, not deduped. JSON parsers silently keep the last duplicate, so the
old config worked; this diff turns off type-aware lint for the whole repo. That's
a behavior change buried in a "feature" PR with no mention in the body.

**(b) `selfReflectPrompt ?? buildAgentPrompt` substitution.** The new `build`
agent always receives a non-undefined `prompt` field (either the self-reflect
or the verification text), but `agent.ts:130` is the *only* place this string
is used — which raises a question the diff doesn't answer: does setting `.prompt`
on the built-in `build` agent compose with or replace the existing system prompt?
If it replaces, this PR has just clobbered the entire build-agent system prompt
with a 4-line "Verification" reminder, which would be catastrophic. If it
composes, fine — but the change should include a test.

## Specific line refs

- `packages/opencode/src/agent/agent.ts:109-114` — the new `selfReflectPrompt` /
  `buildAgentPrompt` declarations.
- `packages/opencode/src/agent/agent.ts:130` — `prompt: selfReflectPrompt ?? buildAgentPrompt`
  on the `build` agent (the substitution-vs-composition question above).
- `packages/opencode/src/agent/agent.ts:242-265` — `review` subagent block.
- `packages/opencode/src/agent/agent.ts:266-289` — `test` subagent block.
- `packages/opencode/src/agent/prompt/review.txt:1-27` — new prompt; "Read-only
  analysis only. No code changes." matches the deny-all permission posture.
- `packages/opencode/src/agent/prompt/test.txt:1-30` — new prompt; allows
  `edit`/`write`, prompt should explicitly remind the agent to scope writes to
  test files only.
- `.oxlintrc.json:44-49` (deleted) — `typeAware: true` deleted from lint config
  with no justification in the PR body.
- `packages/opencode/src/agent/prompt/explore.txt` — entire rewrite; functionally
  equivalent but should be its own PR.

## Reasoning

Three independently-reviewable changes plus one undocumented behavior change
(lint config) packaged as one large PR titled "enhance agent capabilities."
The unrelated changes plus the lint-config side-effect plus the unverified
build-prompt substitution semantics push this firmly into "needs work before
merge" territory. Individually each piece is reasonable:

- The `review` subagent design is solid — read-only permissions match its job,
  the `external_directory` allow-list pattern is consistent with how `explore`
  is gated, and the prompt's severity-tier output format is genuinely useful
  for a delegated review.
- The `test` subagent's permission model is right (write-capable, since it
  needs to add test files), but the prompt should say "only modify files under
  `**/*.test.*` or `**/test/**` directories" or similar — otherwise the agent
  has free rein to "fix" production code under the guise of "improving tests."
- The `self_reflect` experimental flag is fine in concept, but the
  `?? buildAgentPrompt` fallback means the cfg flag *adds* the section
  versus *omitting* it, which is the opposite of how an "experimental"
  opt-in flag usually works (you'd expect default = no extra prompt section).
- The `explore` rewrite removes the explicit "do not create any files" line
  from the prompt body — `explore` already has read-only permissions so the
  prompt removal isn't load-bearing, but defense-in-depth is worth keeping.

The PR body claims "20+ security/reliability fixes" but I count zero in the
diff. Either the body is overstated or the relevant commits got dropped during
rebase — either way the description doesn't match the change.

## Verdict

**request-changes** — split into three PRs: (1) the two new subagents
(`review`+`test`) with their prompts and agent.ts registrations, (2) the
`self_reflect` experimental flag with a documented composition story (does
`.prompt` replace or append to the built-in build prompt? add a test), and
(3) the `explore` prompt rewrite as a standalone polish PR. Restore the
`typeAware: true` option in `.oxlintrc.json` (the duplicate was a JSON-syntax
error, not a behavior intent — drop one copy, keep the option) and add a CI
note about why. In the test-agent prompt at `prompt/test.txt`, add an
explicit "scope edits to test files only — do not modify implementation
under test" line, because subagent prompts are the only check between a
write-capable agent and arbitrary repo edits. Update the PR body to match
the actual diff (no "20+ fixes") or include them.
