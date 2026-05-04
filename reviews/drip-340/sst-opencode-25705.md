# Review: sst/opencode #25705 — feat: add Create Folder button to Web UI homepage

- **PR**: https://github.com/sst/opencode/pull/25705
- **Author**: VuKrampHub
- **Base**: `dev`
- **Head SHA**: `67c105a90bc474a957e615284830cbf6b7a2565d`
- **Size**: +221 / −9 across 5 files
- **Closes**: #21255

## Scope

Adds a "New Folder" entry point on the Web UI homepage so users can create project directories from the browser. New `POST /global/file/mkdir` server endpoint, a new `<DialogCreateFolder>` Solid component, and wiring in `home.tsx` / `layout.tsx` plus an `en` i18n key bundle.

## Substantive review

### Backend — `packages/opencode/src/server/routes/global.ts` (+37)
- Adds a `POST /global/file/mkdir` route accepting `{ path: string }` and calling `mkdir(path, { recursive: true })`.
- Returns the created path on success and a JSON error string on failure.
- **Security concerns**: the diff (as seen in this PR) does not show any path validation. `recursive: true` plus an unconstrained absolute path means any caller of the local API can create arbitrary directories anywhere the opencode process can write — including under `~/.ssh`, `/tmp/<symlink-target>`, etc. Even though the local API surface is loopback-bound, other endpoints in this file likely already constrain operations to a workspace or `~/`. This route should at minimum:
  1. Resolve the requested path against `os.homedir()` (or the configured roots from `Global.path()`), reject anything that escapes after resolution, and reject paths that traverse a symlink leaving the allowed root.
  2. Reject paths containing `..` segments before resolution rather than relying on resolution alone.
  3. Surface a structured error (`{ code, message }`) instead of a bare `{ error: string }` so the UI can localise.

### Frontend — `packages/app/src/components/dialog-create-folder.tsx` (+118 new file)
- The `INVALID_CHARS = /[<>:"|?*\x00-\x1F]/` regex (line ~28) intentionally omits `/` and `\` because `parentDir + name` is joined with `/`. That is fine for POSIX but on Windows hosts where the server runs natively, `\` should also be rejected from `name`.
- `joinPath(base, name)` (line ~13) strips trailing slashes and concatenates. It does not normalise `..` in `name`. Combined with the lack of server-side validation this is the main escape vector — a user could type `../../etc` and the backend would happily `mkdir -p` it.
- The `parentDir` defaults to `home()` (line ~25), but the component allows arbitrary editing. The dialog should probably show a directory picker constrained to the workspace rather than a free-form text input.
- Error handling at line ~48 (`if (!res.ok || data.error) { setError(data.error ?? language.t(...)) }`) renders the raw server `error` string into the DOM. Solid auto-escapes text bindings, so this is XSS-safe, but it does leak unfiltered server error text to the user — fine for an internal local API, but a localised generic message would be preferable.
- `creating()` flag is set/cleared but the button is not visually disabled in the diff snippet I have — please confirm there is a `disabled={creating()}` on the submit button to prevent double-clicks creating duplicate dirs (not strictly a bug given idempotent `mkdir -p`, but still worth it).

### Pages — `packages/app/src/pages/home.tsx` (+35 / −9), `layout.tsx` (+21)
- Adds the "New Folder" button into the homepage layout and registers the dialog at the layout level. Standard wiring.

### i18n — `packages/app/src/i18n/en.ts` (+10)
- Adds keys `dialog.createFolder.{title,parentLabel,nameLabel,error.empty,error.invalid,error.failed,...}`. Only `en` is updated; other locales will fall back. Acceptable for an initial PR.

## Verdict

**request-changes** — the feature is well-structured and the UX is reasonable, but the missing server-side path containment is a real local-API hardening miss: a hostile or careless input can create directories anywhere the opencode process can write. Add an allow-rooted-path check (resolve + `startsWith(allowedRoot)` + reject symlink escape) on the server before merging, and reject `..` in the client-side name validator. Once those are in, this is straightforward to land.
