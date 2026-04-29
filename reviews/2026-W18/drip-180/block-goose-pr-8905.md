# block/goose#8905 — Fix Gemini OAuth onboarding failure

- **PR:** https://github.com/block/goose/pull/8905
- **Author:** @angiejones
- **Head SHA:** `df5e14bd` (full: `df5e14bdcffe87cb1c6a07550ed8d54148d78c44`)
- **Size:** +21/-2 across 2 files

## What this does

Fixes a silent failure in the Gemini Code Assist onboarding flow. The
`loadCodeAssist` API returns `allowedTiers` for new users (not
`onboardTiers`); the existing code only consulted `onboard_tiers` and on
miss fell back to a hardcoded tier id `"FREE"` — which `onboardUser`
rejects with an invalid-argument error. The browser-side OAuth would
succeed and redirect back to goose, then the provider would silently fail
to register itself, leaving the user with "OAuth completed" but no Gemini
in the dropdown.

The fix touches two files:

1. **`crates/goose/src/providers/gemini_oauth.rs`** (+13/-1).
   `LoadCodeAssistResponse` gains an `allowed_tiers: Option<Vec<TierInfo>>`
   field (line 306) and `TierInfo` gets `#[serde(rename_all = "camelCase")]`
   plus a new `is_default: Option<bool>` field. The fallback chain in
   `setup_code_assist` (line 428-439) now reads:

   ```rust
   onboard_tiers
       .first()              // existing path
       .or_else(|| {
           allowed_tiers
               .iter().find(|t| t.is_default.unwrap_or(false))   // new: prefer default
               .or_else(|| allowed_tiers.first())                 // new: any tier
               .and_then(|t| t.id.clone())
       })
       .unwrap_or_else(|| "free-tier".to_string())                // new fallback string
   ```

   Two real changes there: (a) new `allowed_tiers` parse + lookup,
   (b) the literal final fallback flips from `"FREE"` to `"free-tier"`,
   which is the actual id the Code Assist API accepts.

2. **`ui/desktop/src/components/settings/providers/modal/ProviderConfigurationModal.tsx`**
   (+8/-1). The `configureProviderOauth` call now captures its
   `oauthResult` and surfaces the error if present, instead of throwing
   away the result and proceeding to the model picker as if everything
   worked:

   ```ts
   const oauthResult = await configureProviderOauth({ path: { name: provider.name } });
   if (oauthResult.error) {
     const err = oauthResult.error as Record<string, unknown>;
     const errDetail = typeof oauthResult.error === 'string'
       ? oauthResult.error
       : (err?.message as string) ?? (err?.detail as string) ?? JSON.stringify(oauthResult.error);
     throw new Error(errDetail);
   }
   ```

## Why this is the right shape

The two-layer fix is correct: the Rust side fixes the *cause* (parse
`allowed_tiers`, prefer the API's marked default tier, use the actual id
string the API accepts), and the TypeScript side closes the *amplifier*
(silent error swallow that turned every backend failure into "OAuth
worked, you just don't see the provider"). Either fix alone would leave
half the bug in place: without the Rust fix, the new error surface in
the modal would now show users an error they can't act on; without the
modal fix, future regressions in `setup_code_assist` would again be
invisible.

The `is_default` preference is also subtly important — for users with
multiple eligible tiers, picking the API-marked default is more correct
than `.first()` ordering, which depends on response stability.

## Concerns

1. **No test coverage.** Neither
   `crates/goose/src/providers/gemini_oauth.rs` nor the modal change
   gain a test. For the Rust side, a unit test that constructs a
   `LoadCodeAssistResponse` JSON payload (the actual shape the Code
   Assist API returns for a new user) and asserts the resolved
   `tier_id` would lock the regression in. Right now the only signal
   that the fix works is the manual repro the author did. Worth
   asking for at least one happy-path + one `allowed_tiers`-only path
   test.

2. **The `"free-tier"` literal.** Hardcoded fallback strings are
   exactly what got us here ("FREE" was wrong, now we're confident
   "free-tier" is right because the author tested it manually). If
   the Code Assist API ever renames this again, we'll be back. Worth
   a comment at the literal: `// Verified accepted by Code Assist
   onboardUser as of 2026-04. Update via #<issue> if it stops being
   accepted.`

3. **Error message UX.** The new throw in
   `ProviderConfigurationModal.tsx` produces an `Error` whose message
   could be the JSON-stringified raw error object — that's
   user-hostile. Better: surface a friendly "Gemini OAuth failed:
   {short reason}. Check logs for details." and put the
   `JSON.stringify` only into the dev console / logging channel.

4. **`onboard_tiers` vs `allowed_tiers` precedence.** The current
   chain prefers `onboard_tiers.first()` over the API-marked default
   in `allowed_tiers`. That's a fine default but it might be backwards
   for new users — the whole reason `allowed_tiers` is what the API
   returns is precisely *because* `onboard_tiers` is empty. Wouldn't
   change the behavior in the bug case, but it's worth confirming
   that for users where *both* are present (existing-user re-auth?),
   we still get the right tier id.

## Verdict

**`merge-after-nits`** — addresses a real silent-failure mode with the
correct two-layer fix (cause + amplifier). Just wants a unit test on
the tier resolution path and a friendlier modal error message before
it's done. Without those it's still a strict improvement over the
status quo.

## Nits

- Add a Rust unit test for `LoadCodeAssistResponse` parsing +
  `setup_code_assist` tier resolution covering the new
  `allowed_tiers` path.
- Comment the `"free-tier"` literal with the verification date.
- Render a friendlier error string in the modal; reserve
  `JSON.stringify` for the log channel.
