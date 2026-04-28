# block/goose#8879 — refactor: agent provider to use explicit type states

- **Repo**: block/goose
- **PR**: [#8879](https://github.com/block/goose/pull/8879)
- **Author**: kalvinnchau (Kalvin C)
- **Head SHA**: `c5342333c38c371d97ac3b217afee21dd93a5586`
- **Reviewed**: 2026-04-29
- **Verdict**: `merge-after-nits`

## Context

`ui/goose2/src/features/settings/ui/AgentProviderCard.tsx` renders the
"install / sign in" card for an agent provider. Previously, install and
auth state were each stored as `boolean | null`, with `null` overloaded
to mean either "not yet checked" or "no info available". Several
distinct UX states had to be inferred from combinations of those
nullable booleans — including the bug the PR description calls out:
the sign-in button could appear during the "auth status still
checking" window.

The PR replaces both nullable-boolean states with explicit string
unions and reroutes every consumer through the new states.

## What changed

### Type-state declarations (top of file)

```ts
type SetupPhase = "idle" | "checking" | "installing" | "authenticating";  // existed
type InstallStatus = "checking" | "installed" | "missing";                 // new
type AuthStatus = "checking" | "authenticated" | "unauthenticated" | "unknown";  // new
```

### State init keyed on what's actually known

```ts
const [installStatus, setInstallStatus] = useState<InstallStatus>(
  hasBinary && !isBuiltIn ? "checking" : "installed",
);
const [authStatus, setAuthStatus] = useState<AuthStatus>(
  provider.authStatusCommand && hasBinary && !isBuiltIn
    ? "checking"
    : "unknown",
);
```

This is the most important change in the diff: built-in providers and
providers without a `binaryName` start at `installed`/`unknown`
*immediately* instead of going through a spurious `checking` phase
they'd never leave. Same for providers without an `authStatusCommand`
— `authStatus = "unknown"` from the start, never `"checking"`.

### Consumer rewrites

Every site that used to read `isInstalled === true` /
`isInstalled === false` / `isInstalled === null` now reads
`installStatus === "installed"` / `"missing"` / `"checking"`. Same
for `isAuthenticated`. Notably the `needsAuth` derivation:

```ts
const needsAuth =
  installStatus === "installed" &&
  hasAuthCommand &&
  authStatus !== "checking" &&
  authStatus !== "authenticated";
```

This is exactly the bug the PR title calls out: under the old code,
`isAuthenticated === null` was treated the same as
`isAuthenticated === false`, so `needsAuth` (and thus the sign-in
button) was true during the checking window. The new condition
explicitly excludes `"checking"`, so the button stays hidden until
the check resolves.

### Bonus correctness fix

```ts
const installed = await checkAgentInstalled(provider.id);
```

Previously `checkAgentInstalled(provider.binaryName)`. The post-install
verification now keys on provider id, not binary name — the right
identifier for "did this provider install successfully". This also
appears to be the fix referenced in the PR summary as "fix
post-install verification to check installation by provider id".

### i18n moves

Hardcoded English strings (`"Setup failed"`, `"Checking..."`,
`"Installing X..."`, the install-finished-but-CLI-not-on-PATH error,
plus the install/auth phase labels) now go through `t(...)` keys
rooted at `providers.agents.*`. Tested via `pnpm check:i18n`.

### Output dedup

`renderSetupOutput(scrollToEnd)` extracts the previously-duplicated
`<div className="…font-mono…">{setupOutput.map(...)}</div>` block.
Two callers, identical markup before, identical markup after. Pure
cleanup.

## Things I'd want changed before merge

1. **`needsAuth` doesn't handle `unknown`**. Read the condition
   carefully:
   ```ts
   const needsAuth = installStatus === "installed"
                  && hasAuthCommand
                  && authStatus !== "checking"
                  && authStatus !== "authenticated";
   ```
   For a provider without an `authStatusCommand`, `authStatus`
   starts at `"unknown"`. That satisfies all four conditions
   (`!=="checking"`, `!=="authenticated"`), so `needsAuth` is
   `true` — i.e. the sign-in button appears for providers we
   genuinely have no way to check. Is that the intended behavior?
   Plausibly yes (the user might still want to sign in), but
   this differs from the old behavior, where
   `isAuthenticated === null` made `needsAuth` false. Worth
   confirming this is the intended UX shift, and if so adding
   a comment noting it.

2. **`AuthStatus = "unknown"` is overloaded**. It means both
   "the provider has no `authStatusCommand` so we can't check" and
   "we tried, the install failed, so auth status is irrelevant".
   The two cases produce the same UI behavior today, but if they
   ever need to diverge, you'll be back to inferring intent from
   the (`installStatus`, `authStatus`) pair. Cheap fix: split into
   `"uncheckable"` and `"unknown"`. Not blocking.

3. **The `setSetupError` call dropped its English literal but the
   replacement key
   (`"providers.agents.errors.installVerificationFailed"`) needs
   to actually exist in the i18n bundle**. The PR ran
   `pnpm check:i18n` which presumably catches missing keys —
   confirm that pass is in the CI surface for this PR.

## Risk analysis

- **State init changes for built-in providers**: previously they'd
  show "Checking..." briefly during mount even though there was
  nothing to check. New init skips that, so they jump straight to
  the "ready" state. Pure UX improvement.
- **`checkAgentInstalled(provider.id)` swap**: if any provider had
  an id that didn't match its binary name, the verification
  previously checked the wrong target. After this PR, verification
  uses the same id as the original `useEffect` mount-time check
  (also `provider.id`). So this swap actually makes the two
  check sites consistent.
- **i18n migration**: any locale bundle missing the new keys will
  render the key string instead of localized text. Standard i18n
  risk; CI guard handles it.
- **`disabled` prop removed**: the install/connect button used to
  be `disabled={isInstalled === null && hasBinary}`. Removed in
  this PR. Now the button is always enabled — clicking during
  `installStatus === "checking"` will fire `handleConnect` against
  an unknown install state. The handler reads `installStatus` and
  branches correctly (only runs install if status is `"missing"`),
  so the worst case is "click during check → handler does nothing
  visible". Safe but worth a tiny UX note: showing a disabled
  state during the (typically very short) checking window felt
  more polished. Either restore the `disabled` for that window or
  document that the click is a no-op.

## Verdict

`merge-after-nits`. The type-state refactor is the right shape,
genuinely fixes the "sign-in button appears during checking"
race, and the bonus `provider.id` consistency fix is a real
correctness improvement. Two design questions worth answering
inline (the `needsAuth` semantics for `"unknown"` and whether the
`disabled` removal is intentional) before merge — but the diff
direction is unambiguously correct.

## What I learned

`boolean | null` is one of the most-overloaded type pairs in UI
state. The two truthy values usually mean "yes/no" but `null`
often means three things at once: "not checked yet", "no info
available", "intentionally cleared". Once you find yourself
writing `if (x === null && hasFoo) { ... }` and `if (x !== true)
{ ... }` in the same component, the right move is to spell out
the actual states as a string union. The verbosity pays for itself
the first time you have to add a fourth state.
