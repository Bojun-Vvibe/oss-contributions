# block/goose#9047 — move settings into app shell

- PR: https://github.com/block/goose/pull/9047
- Head SHA: `d2b820b0f657b1dea83307b3f270c72eb15c8c23`
- Size: +478/-377 (net +101, mostly restructure)
- Verdict: **merge-after-nits** (man)

## Shape

Refactors goose2's settings UI from a modal-based component
(`SettingsModal`) into an in-shell view that participates in the app's
top-level routing. Net effect: settings get a real URL (`/settings?section=
<id>`), survive page reloads, can be deep-linked, and integrate with the
browser back button.

Key moves in `ui/goose2/src/app/AppShell.tsx`:

1. The hardcoded `SETTINGS_SECTIONS = new Set<SectionId>([...])` literal at
   `:58-68` is deleted in favor of importing `DEFAULT_SETTINGS_SECTION` and
   `isSettingsSection` from a new `settingsSections` module
   (`@/features/settings/ui/settingsSections`). Good — single source of truth
   for the section enum.
2. New `AppView` variant `"settings"` added at `:47` (sibling to `"home" |
   "extensions" | "agents" | "projects" | "session-history"`).
3. URL state plumbing via three new helpers at `:75-100`:
   - `getInitialSettingsSection()` reads `window.location.pathname ===
     "/settings"` and `?section=<id>`, validating against `isSettingsSection`
     before falling back to `DEFAULT_SETTINGS_SECTION`.
   - `setSettingsSectionUrl(section)` uses `history.replaceState` (not
     `pushState`) — correct for "section changes inside settings shouldn't
     stack history entries the user has to back-button through".
   - `clearSettingsSectionUrl()` resets pathname to `/` and removes the
     `section` param — invoked when leaving the settings view.
4. `lastNonSettingsViewRef = useRef<AppView>("home")` at `:149` plus the
   `useEffect` at `:215-220` tracks the previously-active view so when the
   user closes settings, they return to where they were rather than always to
   `home`. This is the right UX shape.

## Notable observations

- `getInitialSettingsSection()` correctly guards `typeof window === "undefined"`
  for SSR safety at `:76` — even though goose2 is electron-only today, the
  guard is cheap and protects against future reuse.
- The `?section=` query param is validated via `isSettingsSection(section)`
  before use at `:81` — preventing URL-injection of invalid section IDs that
  would silently render nothing. Defensible.
- `useState<SectionId>(initialSettingsSection ?? DEFAULT_SETTINGS_SECTION)` at
  `:128` — note `initialSettingsSection` is computed *before* state init, but
  since `getInitialSettingsSection` is a pure function with no React hooks,
  this is fine. Could also be a `useState(() => getInitialSettingsSection() ??
  DEFAULT_SETTINGS_SECTION)` lazy initializer to avoid recomputing on every
  render — micro-optimization, but matches the React idiom.

## Concerns / nits

- `setSettingsSectionUrl` and `clearSettingsSectionUrl` both manipulate
  `window.history.replaceState` — verify electron-side history navigation
  doesn't have any custom intercept that would conflict (e.g. a custom
  protocol handler that breaks on `replaceState` to `/settings`).
- Pulling the `SETTINGS_SECTIONS` set out of `AppShell.tsx` and into
  `settingsSections.ts` is the right move, but I can't see the new module's
  source in the visible diff — confirm `isSettingsSection(section: string):
  section is SectionId` is implemented as a `Set.has` check (O(1)) not an
  array `.includes` (O(n)) on every URL parse.
- No test in the visible diff for the URL-restore-on-reload contract or for
  the "back to previous view" ref — both are integration-level behaviors
  worth pinning with a smoke test.
- `lastNonSettingsViewRef` defaulting to `"home"` at `:149` means a user who
  *deep-links* to `/settings?section=appearance` and then closes settings
  lands on `home` — fine, but should be documented as the intentional
  fallback (not a bug to be "fixed" later by treating `null` as "stay where
  you are").
