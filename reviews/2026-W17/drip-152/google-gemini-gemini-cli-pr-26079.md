# google-gemini/gemini-cli#26079 — fix(core): better error message for failed cloudshell-gca auth

- **Repo:** [google-gemini/gemini-cli](https://github.com/google-gemini/gemini-cli)
- **PR:** [#26079](https://github.com/google-gemini/gemini-cli/pull/26079)
- **Head SHA:** `fd58991c82b09d704145f7aa9cb29521b3cd7c5c`
- **Size:** +44 / -0 across 2 files (`server.test.ts`, `server.ts`)
- **State:** MERGED (addresses #25946)

## Context

Cloud Shell users hitting the default `cloudshell-gca` Gemini project
sometimes get 403 Permission Denied responses from `loadCodeAssist`
when their environment isn't actually entitled to the shared default.
The CLI was surfacing the raw 403 to users, with no actionable
recovery path — leaving Cloud Shell users staring at a permission
error with no idea they could resolve it by setting their own
GOOGLE_CLOUD_PROJECT.

## Design analysis

Two surgical additions:

1. **403 detector helper** at `server.ts:587-595`:

   ```ts
   function isPermissionDeniedError(error: unknown): boolean {
     return (
       !!error &&
       typeof error === 'object' &&
       'response' in error &&
       !!error.response &&
       typeof error.response === 'object' &&
       'status' in error.response &&
       error.response.status === 403
     );
   }
   ```

   Defensive shape — every property access is guarded by `in`-operator
   plus typeof check, so a malformed error object doesn't throw
   inside the discriminator. Right pattern for `unknown`-typed inputs:
   walk the shape with explicit narrowing rather than rely on type
   assertions. Returns `false` on any malformed input which means the
   caller falls through to the generic re-throw at `server.ts:284`
   — that's the right default (don't swallow errors you don't
   recognize).

2. **Targeted error rewrite** at `server.ts:276-283` in
   `loadCodeAssist`'s catch block:

   ```ts
   } else if (
     isPermissionDeniedError(e) &&
     req.cloudaicompanionProject === 'cloudshell-gca'
   ) {
     throw new Error(
       'Access to the default Cloud Shell Gemini project was denied.\n' +
         'Please set your own Google Cloud project by running:\n' +
         'gcloud config set project [PROJECT_ID]\n' +
         'or setting export GOOGLE_CLOUD_PROJECT=...',
     );
   }
   ```

   The discriminator is a *conjunction* of two conditions: 403 status
   AND the request was for the default cloudshell-gca project. The
   conjunction is critical — a 403 on a user's *own* project is an
   actually-different problem (they need IAM, not a project switch),
   so the recovery instruction would be wrong-and-confusing if the
   gate fired on every 403.

   Placement *before* the generic `throw e` at `:284` is the right
   ordering — the new branch handles the specific recognized case,
   else-fall-through preserves the existing generic-error path.

## Test coverage

Single new test at `server.test.ts:647-666`:

```ts
it('should throw friendly error for 403 on cloudshell-gca project', async () => {
  const { server } = createTestServer();
  const mock403Error = {
    response: { status: 403, data: { error: { message: 'Permission denied' } } },
  };
  vi.spyOn(server, 'requestPost').mockRejectedValue(mock403Error);

  await expect(
    server.loadCodeAssist({
      cloudaicompanionProject: 'cloudshell-gca',
      metadata: {},
    }),
  ).rejects.toThrow(/Access to the default Cloud Shell Gemini project/);
});
```

Asserts the regex pattern matches the new friendly message. The
test correctly mocks the *shape* of the actual error returned by
the underlying HTTP client (which is what `isPermissionDeniedError`
walks), not just a generic `Error` object.

## Risks / nits

1. **No negative test for the symmetric cases.** The PR adds one
   positive test (`cloudshell-gca` + 403 → friendly message) but
   no negative tests for:
   - 403 on a user's own project → original error preserved
   - 401 on cloudshell-gca → original error preserved (auth, not
     permission)
   - non-cloudshell-gca + 403 → original error preserved

   These are the exact cases the conjunction in the discriminator
   is supposed to *not* match. Without negative tests, a future
   refactor that loosens the conjunction (e.g., dropping the
   project-name check) would silently regress to "all 403s get the
   cloudshell message" which is wrong-and-confusing for normal
   IAM-misconfigured users.

2. **Hard-coded `cloudshell-gca` project name is policy-as-string.**
   `server.ts:280` checks `req.cloudaicompanionProject ===
   'cloudshell-gca'` against a literal. If Google ever renames the
   default Cloud Shell project (or adds a regional variant like
   `cloudshell-gca-eu`), the gate misses the new project. Worth
   extracting to a named constant (`CLOUDSHELL_GCA_DEFAULT_PROJECT_ID`)
   so the dependency is grep-able from one place.

3. **Error message hardcodes `[PROJECT_ID]` as a placeholder.** The
   message reads `gcloud config set project [PROJECT_ID]` — square
   brackets indicate placeholder per gcloud convention, but novice
   users may copy-paste literally. Worth changing to
   `<your-project-id>` or `YOUR_PROJECT_ID` (more visually obvious
   as a placeholder) plus an example like `gcloud config set
   project my-project-123`.

4. **No log/telemetry signal for the cloudshell-gca-403 case.** The
   error is re-thrown (so it'll propagate to the user), but there's
   no debug log capturing "we hit the cloudshell-gca-default-403
   path." That signal would be useful for the gemini-cli team to
   measure how many users hit this case and whether the friendly
   message reduces support tickets. Worth a `debugLogger.debug()`
   line.

5. **`isPermissionDeniedError` is unexported but defensively typed
   as if it might be reused.** The walker shape (`unknown` →
   `boolean` with full guards) suggests reuse, but it's only called
   once. Either inline it (small enough at 9 lines) or export it
   for the broader `code_assist/` module to share — the in-between
   state of "defensively-shaped private helper called once" is the
   classic shape that gets duplicated next time someone needs the
   same check elsewhere.

## Verdict

**merge-after-nits.** The fix is minimal, well-scoped, and the
discriminator conjunction is correctly tight. Pre-merge ask: nit 1
(add at least the negative test for "403 on user's own project →
generic error preserved"). Nit 2 (extract project-name constant)
is a minor refactor. Nits 3-5 are post-merge.

## What I learned

- "Targeted error rewrite" is the right pattern for actionable
  recovery messages — the discriminator should be a *conjunction*
  of every condition that makes the recovery message correct,
  not a disjunction. Looser gates produce wrong advice for
  adjacent error cases.
- `unknown`-typed error walking via `'prop' in error` + `typeof`
  guards is the safe pattern for `catch(e: unknown)` in
  TypeScript — type assertions can crash at runtime if the
  object shape is malformed, the in-operator/typeof walk
  short-circuits to `false` instead.
- Hard-coded magic strings for "this is the default project name"
  type policy are a slow-burn maintenance hazard. Extracting to
  named constants in one location makes future renames a
  single-edit change rather than a grep-and-pray.
- Adding actionable recovery instructions to error messages is
  high-leverage — the same lines of code change a "user can't
  proceed" failure into a "user does the suggested thing and
  recovers" success. Worth pattern-matching for any user-facing
  error message that doesn't tell the user what to do next.
