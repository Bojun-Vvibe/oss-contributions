# BerriAI/litellm#26924 — fix(ui): remove insecure ?token= URL handler from LoginPage to close session-fixation

- Repo: BerriAI/litellm
- PR: https://github.com/BerriAI/litellm/pull/26924
- Head SHA: `05a89b2c92cb`
- Size: +81 / -15 (two files: one source, one test)
- Verdict: **merge-as-is**

## What the diff actually does

Pure removal of an attacker-injectable login path, paired with two
regression-pinning tests.

1. **`ui/litellm-dashboard/src/app/login/LoginPage.tsx:69–80`** — deletes
   the entire 12-line `urlToken` block that previously did:
   ```ts
   const urlToken = params.get("token");
   if (urlToken && !isJwtExpired(urlToken)) {
     document.cookie = `token=${urlToken}; path=/; SameSite=Lax`;
     params.delete("token");
     const cleanSearch = params.toString();
     window.history.replaceState(...);
     router.replace("/ui/?login=success");
     return;
   }
   ```
   The deletion is total — there is no replacement, no feature flag, no
   "warn but keep working" middle ground. The legacy URL flow is gone.

2. **`ui/litellm-dashboard/src/app/login/LoginPage.test.tsx:292–373`** —
   adds a dedicated `describe("URL ?token= legacy path is rejected
   (security regression test)", ...)` block with two tests pinning the
   anti-behavior:
   - **Test 1 (`:316–344`):** seeds `window.location.search =
     "?token=attacker.jwt.value"`, mocks `getCookie` returning `null`
     (no existing session), mocks `isJwtExpired` returning `false` (the
     attacker JWT is structurally valid, so the prior code path WOULD
     have accepted it), renders, and asserts both `document.cookie` does
     **not** contain `token=attacker.jwt.value` AND `mockReplace` was
     **not** called with `/ui/?login=success`. This is the exact-shape
     CVE assertion.
   - **Test 2 (`:346–372`):** same URL setup but `getCookie` returns
     `"legitimate-session-jwt"` (existing valid session). Asserts the
     attacker token never overwrites the legitimate cookie *and* the
     redirect path is `/ui` (the normal already-logged-in flow), not
     `/ui/?login=success` (the post-login flow). This pins the second
     attack arm: session fixation against an already-logged-in user.

## Why merge-as-is

- **Threat model is textbook session fixation.** Any link of the form
  `https://your-litellm-dashboard.example.com/ui/login?token=<attacker_jwt>`
  would, under the prior code, set the attacker's JWT as the session
  cookie if the user clicked the link. The deletion closes the entire
  attack surface — there is no alternative way to set a token from a URL
  parameter after this PR.
- **No legitimate usage to preserve.** The deleted block was labeled
  "Backwards compat" with no version pin and no documented call site.
  SSO flows use the cookie-set-by-server pattern (visible in the rest of
  the file), and the `/ui/onboarding` token-acceptance flow lives on a
  different page. The only "users" of `?token=` were either nobody or
  attackers.
- **Tests assert the anti-behavior, not the implementation.** Test 1
  doesn't check that the `urlToken` block was removed; it checks that
  *no path through the component* sets the cookie or triggers the
  `?login=success` redirect when `?token=` is in the URL. A future
  contributor who re-introduces a URL-token handler under a different
  variable name will break the test.
- **Test 2 closes the silent-overwrite arm.** Even if a future
  contributor argued "we should accept ?token= when no session exists,"
  Test 2 separately pins that an existing valid cookie is never
  overwritten, which is the harder-to-spot attack arm (the user is
  already logged in and the URL parameter silently swaps their session).
- **`isJwtExpired` returning `false` in Test 1** is the load-bearing
  setup detail — it ensures the prior code path *would have accepted*
  the attacker token, so the test is genuinely asserting the security
  property and not just relying on JWT-expiration as the safety net.
- **`beforeEach`/`afterEach` properly restore `window.location`** via
  `Object.defineProperty(... originalLocation)`, so the tests don't
  contaminate sibling tests in the file.

## Nits I would NOT block on

- The cookie cleanup in `beforeEach` (`document.cookie = "token=; expires=Thu, 01 Jan 1970 ..."`)
  is correct boilerplate but wouldn't hurt to extract into a
  `clearCookie(name)` helper if the file grows more cookie-stateful
  tests.
- The PR body does not name a CVE ID (per body: "internal security audit
  finding (automated scanner)"), which means downstream consumers
  parsing changelogs for CVE refs won't find one — fine for an internal
  finding, but a CHANGELOG note like "(security: fixes session
  fixation in dashboard login)" in the merge commit would help operators
  grepping release notes.

## Theme tie-in

Fix the auth-bypass at the *intent layer* (delete the entire
URL-parameter handler) rather than at the *value layer* (validate the
JWT more carefully). Validation-layer fixes leave the attack surface
open to future bypass; intent-layer deletion closes it permanently. The
two regression tests pin the anti-behavior so a future contributor
can't silently re-introduce the surface under a different variable
name.
