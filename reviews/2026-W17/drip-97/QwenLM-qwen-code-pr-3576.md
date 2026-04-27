# QwenLM/qwen-code#3576: Feat/openrouter auth

- **Author:** pomelo-nwu (pomelo)
- **HEAD SHA:** d7ef4565
- **Verdict:** merge-after-nits

## Summary

A 5519/-120 PR that lands two related things: (1) browser-based
OpenRouter OAuth (PKCE flow with localhost callback listener) wired
into both `qwen auth openrouter` CLI subcommand and the interactive
`/auth` TUI dialog, persisting the exchanged API key under
`env.OPENROUTER_API_KEY` and selecting `AuthType.USE_OPENAI`; and
(2) a new `/manage-models` interactive dialog that fetches the
OpenRouter `/api/v1/models` catalog, filters out non-text-output
models, applies a curated recommended-default enabled subset, and
exposes search/filter/multi-select catalog management with persist
to `settings.modelProviders.openai`.

The design rationale is documented up front in
`docs/design/openrouter-auth-and-models.md` (+89 lines net new), and
the central architectural choice — separating *catalog* (full
discovered set) from *enabled set* (what shows in `/model`) — is
correct and avoids the obvious failure mode of dumping hundreds of
OpenRouter models into the user's settings on first auth. Reusing
`AuthType.USE_OPENAI` instead of inventing a new auth type is the
right call given OpenRouter is already OpenAI-compatible at the
wire level, and the PR explicitly preserves non-OpenRouter
OpenAI-compatible provider entries when updating the OpenRouter
enabled set.

## Specific feedback

- `packages/cli/src/commands/auth/openrouterOAuth.ts` (+723/-0) is
  the central new file. The handler at
  `packages/cli/src/commands/auth/handler.ts:295-340` correctly
  threads `oauthSession.authorizationUrl` into the user-facing
  prompt before opening the browser, which is the right UX for
  remote/SSH sessions where browser auto-launch may fail. Verify
  `runOpenRouterOAuthLogin` actually surfaces the localhost-bind
  failure (port-in-use) as a recoverable error, not a process
  abort.
- `openrouterOAuth.test.ts` (+655) and `openrouter.test.ts` (+304)
  give a substantive test surface, but the PR body explicitly
  notes the targeted `vitest` run failed in the review worktree
  due to missing `undici` and `@testing-library/react` deps. CI
  should be the source of truth here — confirm green before merge.
- `packages/cli/src/ui/components/ManageModelsDialog.tsx` (+648/-0)
  is the largest UI addition. The "filter out non-text-output
  models" logic should be defensive against future OpenRouter
  catalog schema changes — if `architecture.output_modalities`
  is missing from a future model, the safe default should be
  "include with a warning" not "silently drop", to avoid users
  thinking models disappeared. Worth a brief inline comment on
  the chosen policy.
- i18n update: 8 locale files touched (+2 each:
  `de.js`/`en.js`/`fr.js`/`ja.js`/`pt.js`/`ru.js`/`zh-TW.js`/`zh.js`)
  — confirm all 8 locales got the same key added. If any locale
  lacks the OAuth-provider description string the dialog will
  fall back to the key name, which is ugly but not broken.
- `packages/cli/src/commands/auth/handler.ts:14-26` reorders an
  import: the `getCodingPlanConfig`/`isCodingPlanConfig`/
  `CodingPlanRegion`/`CODING_PLAN_ENV_KEY` symbols are now imported
  from `'../../constants/codingPlan.js'` instead of the
  `@qwen-code/qwen-code-core` package re-export. Confirm this isn't
  a regression that breaks external consumers who imported these
  same symbols from the public `core` package.
- `.gitignore` adds `tmp/` without a trailing newline — minor
  hygiene nit, every other line ends in `\n`.
- Recommended default enabled subset: the design doc says "small,
  stable, biased toward models that let users try the product
  successfully, including free models when available" but the
  actual list isn't pinned in the design doc. Worth listing the
  exact recommended set in the doc so the policy is auditable.
- `useAuth.ts` (+345/-1) and `AuthDialog.tsx` (+620/-33): big UI
  rewrites. The cancellation/abort path on the OAuth flow
  (mentioned in the PR body) needs at least one explicit test
  asserting the localhost listener is closed and the TUI returns
  to the auth menu cleanly when the user aborts mid-flow.

## Risks / questions

- PKCE-flow security: confirm `code_verifier` is generated with
  a cryptographically secure RNG (Node `crypto.randomBytes` or
  Web `crypto.getRandomValues`), not `Math.random`, and that
  `state` is used and validated on the callback to prevent
  CSRF-style attacks on the localhost listener.
- Localhost callback port: if hardcoded, document it; if random,
  confirm the redirect URI registered with OpenRouter supports
  dynamic ports (most OAuth providers require an exact match).
- Settings migration: existing users who already manually
  configured an OpenRouter entry under `modelProviders.openai`
  should not have their entries clobbered. The PR claims to
  preserve non-OpenRouter entries; an existing OpenRouter entry
  edge case wants explicit test coverage.
- The `qwen auth openrouter --key <key>` flow bypasses OAuth
  entirely — confirm key validation hits OpenRouter
  (`/api/v1/auth/key` or equivalent) before persisting, otherwise
  a typo'd key persists silently and only fails on first request.
