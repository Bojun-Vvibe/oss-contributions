# BerriAI/litellm#26631 — fix(ui): move 'Store Prompts in Spend Logs' toggle to Admin Settings

- **Repo:** [BerriAI/litellm](https://github.com/BerriAI/litellm)
- **PR:** [#26631](https://github.com/BerriAI/litellm/pull/26631)
- **Head SHA:** `7f48284decb155c257445e27f77f98266b74737c`
- **Size:** +532 / -719 across 14 files (`useProxyConfig.ts`, `useStoreRequestInSpendLogs.ts`, `AdminPanel.tsx`, new `LoggingSettings/`, deleted `SpendLogsSettingsModal/`, refactored `view_logs/`)
- **State:** MERGED (resolves LIT-2426)

## Context

The "Store Prompts in Spend Logs" + "Maximum Spend Logs Retention
Period" controls were on the Logs page behind a gear icon. Two real
problems: (1) the gear was visible to *every* authenticated user even
though `/config/update` and `/config/list` both require `PROXY_ADMIN`
— so non-admins saw the gear, opened the modal, and got a 403 on load
and another on save. UX-wise this is the worst kind of broken: the
button exists, the form opens, then the form silently fails. (2) These
are *global* proxy settings, not per-user logs-page settings, so the
visual placement was misleading even for admins. Both fixed by moving
to the Admin Settings tab tree (already gated to admins at the sidebar
via `all_admin_roles`).

## Design analysis

Three correctness improvements bundled cleanly:

1. **React Query cache invalidation on mutation success** at
   `useProxyConfig.ts:170-184`. The `useDeleteProxyConfigField` mutation
   gains an `onSuccess: () => queryClient.invalidateQueries({ queryKey:
   proxyConfigKeys.all })` callback, and the `proxyConfigKeys` factory
   is exported so the symmetric write-side hook
   (`useStoreRequestInSpendLogs.ts:54-65`) can invalidate the same
   keys. Right shape — a global proxy-config mutation that doesn't
   invalidate the read cache leaves the UI showing stale data until a
   manual refresh, exactly the "I just saved this, why doesn't it show
   up" footgun. The shared keys factory at `:104` (`export const
   proxyConfigKeys = createQueryKeys("proxyConfig")`) is the right way
   to keep the write-side and read-side coupled — without the export
   they'd drift to two different key strings and the invalidation
   would silently no-op.

2. **Test for the cache-invalidation contract** at
   `useProxyConfig.test.ts:430-450`. The test installs a
   `vi.spyOn(queryClient, "invalidateQueries")`, runs the delete
   mutation, awaits success, then asserts `expect(invalidateSpy)
   .toHaveBeenCalledWith({ queryKey: proxyConfigKeys.all })`. This is
   the *only* way to catch a future regression where someone removes
   the `onSuccess` callback or changes the key shape — the
   stale-data UI bug it fixes is invisible to integration tests
   that don't render mid-mutation. The spy-on-queryClient pattern
   is also the right test surface (lower than full RTL, higher than
   unit-testing the callback in isolation).

3. **Modal → Admin Settings tab relocation**. Net `-484/-156 = -640`
   lines from deleting `SpendLogsSettingsModal/` (the modal component
   + its 484-line test suite), `+335/+145 = +480` lines for the new
   `LoggingSettings/` (component + test). Net negative LOC for a
   functionally-equivalent UI is a good sign. The
   `AdminPanel.tsx:24,365` adds `<LoggingSettings />` to the
   admin-tabs tree alongside `SSOSettings`, `UISettings`,
   `HashicorpVault` — uniform with the existing tab pattern.

4. **`mutate` + `onSuccess`/`onError`/`onSettled` instead of
   `mutateAsync` + try/catch** in the new `LoggingSettings`. The PR
   body calls out that the old shape had a double-toast bug because
   both the per-call `onError` (from React Query's hook config) and
   the outer try/catch (from the await wrapper) fired notifications
   on failure. Moving to the callback shape removes the wrapper
   layer, so error notifications fire exactly once. This is a real
   class of bug — `mutateAsync` is a strict footgun when paired with
   per-mutation error callbacks because the rejection path runs both.

5. **Prop-chain cleanup at the call sites**. `ConfigInfoMessage.tsx`,
   `LogDetailContent.tsx`, `LogDetailsDrawer.tsx` all lose their
   `onOpenSettings` prop chain (3-deep callback drilling). The new
   `ConfigInfoMessage` at `:3-14` just inlines the user-facing
   instruction "navigate to Admin Settings → Logging Settings"
   instead of trying to deep-link the user there with a callback.
   Right tradeoff: callback drilling for a once-used prop is worse
   than a static text reference, and the static text doesn't break
   when the admin-tabs tree restructures.

## Risks / nits

1. **Admin-tabs tree growth is unbounded.** This PR adds
   `LoggingSettings` to a list that already includes `SSOSettings`,
   `UISettings`, `HashicorpVault`, and others. There's no organizing
   principle (alphabetical? grouped by concern?), so adding the
   N+1th tab will keep getting harder to find for users. Worth a
   secondary-grouping pass (e.g., "Auth", "Logging & Telemetry",
   "Integrations") before the next addition.

2. **Stale-key drift is structurally possible.** `proxyConfigKeys` is
   exported and consumed from two files
   (`useStoreRequestInSpendLogs.ts:4` imports it from
   `useProxyConfig.ts`), but there's no test asserting both files
   reference the *same* exported value. A future refactor that
   re-exports `proxyConfigKeys` from a barrel could silently break
   one consumer's imports. Worth a barrel re-export or a
   smoke-test asserting `import.meta` equivalence.

3. **`mutate` callback shape loses the await semantics for chained
   mutations.** The PR replaces `await mutateAsync()` chains with
   `mutate({ ..., onSettled: nextStep })` — fine for the simple
   delete-then-update sequencing the PR fixes, but if a future
   form requires three-step chaining the callback nesting will
   become harder to read than the await chain was. Worth keeping
   the `mutateAsync` pattern in mind for genuinely-multi-step flows
   even though it's wrong here.

4. **Test for the non-admin path is manual-only.** The PR's "test
   plan" section claims "As a non-admin (Internal User), visit the
   Logs page — gear icon is gone" was verified manually. The
   structural fix (the gear is just removed from the JSX) is correct
   but no automated test asserts a non-admin-rendered Logs page has
   no gear. Worth a Storybook visual-regression or RTL test.

5. **No data-migration story for orgs with the old config.** The
   underlying proxy config keys (`store_prompts_in_spend_logs`,
   `maximum_spend_logs_retention_period`) are unchanged — the move
   is purely UI — so existing settings are preserved. But this
   could be called out in the changelog so admins don't worry their
   settings were reset by the relocation.

## Verdict

**merge-after-nits.** The defense-in-depth fix (move the privileged
control behind the admin-only gate) is correct and well-shaped, the
double-toast bug fix is real, and the cache-invalidation correctness
improvement is the kind of fix that prevents long-tail "why doesn't
my UI update" support tickets. Pre-merge ask: nit 5 (changelog note
that no settings are reset). Nits 1-4 are post-merge.

## What I learned

- "Privileged controls should not be visible to unprivileged users
  even if the privileged endpoints are properly gated" is a
  defense-in-depth principle: the UX of "you can see the button
  but it 403s on click" is much worse than "the button is just
  not there." Defense-in-depth at the visibility layer prevents
  user confusion, not just unauthorized access.
- React Query mutations *must* invalidate their read-cache keys on
  success, or the UI shows stale data until a manual refresh. The
  pattern is so common it's worth a lint rule against `useMutation`
  bodies without an `onSuccess` callback that touches `queryClient`.
- `mutateAsync` + outer try/catch is the wrong pattern when the
  mutation is also configured with `onError` callback. Both fire
  on rejection, producing a double-toast bug that's invisible
  until a real backend failure exercises the path. The callback
  shape (`mutate` + `onError`) is the safer default.
- Sharing query-key factories via `export` is the right way to
  couple read-side and write-side hooks for a shared resource.
  The bug-shape of un-shared keys is silent: invalidation no-ops
  because the keys don't match, and the UI stays stale.
