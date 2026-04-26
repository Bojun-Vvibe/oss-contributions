---
pr: 3583
repo: QwenLM/qwen-code
sha: 11ef4771b314ec8f105d544b24a0f06899e0d91f
verdict: needs-discussion
date: 2026-04-27
---

# QwenLM/qwen-code #3583 — docs(auth): add custom API key wizard PRD

- **Author**: pomelo-nwu
- **Head SHA**: 11ef4771b314ec8f105d544b24a0f06899e0d91f
- **Link**: https://github.com/QwenLM/qwen-code/pull/3583
- **Size**: +864/-0 — single new file `docs/design/custom-api-key-auth-wizard-prd.md`.

## Scope

Closes #3582. Pure docs: an 864-line PRD proposing an in-terminal wizard to replace the current "documentation-only screen" that appears when users pick `Custom API Key` from `/auth`. The proposed flow: protocol select → base URL → API key → model IDs → JSON review → save + authenticate, with auto-generated Qwen-managed `envKey` names so users never hand-edit `settings.json` and never collide with shell env vars.

## Specific findings

- Document lives at `docs/design/custom-api-key-auth-wizard-prd.md:1-864`. The repo doesn't currently appear to have a `docs/design/` tree per the diff (single new file). Fine to seed a new tree, but a maintainer should green-light "do we want a `docs/design/` directory at all?" before this lands — the question is editorial, not technical.
- Summary at `:1-15` and Background at `:17-49` are accurate to the current state — the static info screen text quoted at `:21-29` matches what users see today. The `/auth` tree rendered at `:53-69` reflects current UX correctly, and the proposed tree at `:97-119` shows where new states are inserted.
- Goals at `:73-91` — ten goals, all reasonable. Goal 5 ("Automatically generate a Qwen-managed private envKey from the selected protocol and input baseUrl") is the most consequential — it implies the wizard mints a name that lives under `settings.json.env` rather than the user's shell, which is the right call for the "no shell collision" goal (Goal 7) but introduces a new convention that needs explicit naming-scheme documentation in the PRD (currently the PRD says "Qwen-specific generated key name" but doesn't lock down the format — `QWEN_CUSTOM_<HASH(baseUrl)>` vs `QWEN_CUSTOM_<PROTOCOL>_<INDEX>` etc.).
- Supported protocols table at `:130-138`: `openai` / `anthropic` / `gemini` map to `modelProviders` keys directly. The table is correct against current `authType` semantics. **Missing**: the PRD does not address what happens if the user has *already* configured the same protocol via shell env (e.g. `OPENAI_API_KEY` is exported) — does the wizard's generated `envKey` shadow it, or does the existing env still take precedence at runtime? This matters because users who already have `OPENAI_API_KEY` exported and run the wizard for an OpenAI-compatible internal gateway will be surprised either way.
- Non-goals at `:93-99` are solid — explicitly excludes `generationConfig`/`capabilities`/per-model overrides from the wizard, keeps the doc link, doesn't auto-detect protocol. This is the right scoping.
- Target users at `:103-106` — covers OpenAI-compatible / Anthropic-compatible / Gemini-compatible plus vLLM / Ollama / LM Studio / internal gateways. Matches the protocol table.
- The doc is well-structured but quite long for a v1 PRD (864 lines). A maintainer should decide whether the level of detail (review-JSON layouts, example settings-file diffs, error-state copy) belongs in a PRD or in a follow-up implementation doc. Some of this should probably move to `docs/design/<feature>/implementation.md` once the PRD is approved.
- No code, no tests, no risk to runtime — but also no concrete acceptance criteria written as testable predicates. "Authenticate immediately after saving and show success or failure feedback" (Goal 10) needs to specify *what kind* of feedback (toast, blocking dialog, status line) and *what happens on failure* (retry, rollback, leave settings written?).

## Risks

Documentation-only, so no runtime risk. The "should this PR exist at all" question is real — PRDs in-tree are a maintainer-style decision, and once accepted they tend to attract more PRDs that may or may not stay synced with the implementation.

## Verdict

**needs-discussion** — the proposed UX is reasonable and the doc is well-organized, but two things need maintainer alignment before this should land: (1) does the project want PRDs committed to `docs/design/` at all (vs. tracked in issues / a wiki / an external design doc)? and (2) the auto-generated `envKey` naming convention and its shadowing semantics vs existing shell env vars need to be locked down before implementation, not deferred. These aren't quality blockers — they're scope/governance questions that the author can't resolve unilaterally.
