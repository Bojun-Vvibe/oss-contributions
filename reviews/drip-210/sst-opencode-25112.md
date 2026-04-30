# sst/opencode#25112 — feat(cli): add TUI custom provider setup

- PR: https://github.com/sst/opencode/pull/25112
- Head SHA: `166b9ce06407d45aa711107176321f80adc3b7f0`
- Author: yanzhanglee
- Files: 3 changed (`dialog-custom-provider-form.ts` +113, `dialog-custom-provider-form.test.ts` +78, `dialog-provider.tsx` +321/−101), +512/−101

## Context

The TUI provider picker only ever exposed first-party providers from
`sync.data.provider_next.all` (sorted by `PROVIDER_PRIORITY`); users who wanted
an OpenAI-compatible endpoint had to drop into the WebUI custom-provider flow
or hand-edit `opencode.json`. This PR puts that flow inside the TUI as a
"Custom provider" entry pinned at the top of the dialog.

## Design

The diff splits cleanly into a pure-data validator and a UI shell.

**Validator (`dialog-custom-provider-form.ts:51-113`).** `validateCustomProvider`
takes the form, the existing-provider-ID set, and the disabled-provider list,
and returns a tagged-union `CustomProviderValidation` (`{ ok: true, ... } | { ok:
false, error }`). It enforces:

- `PROVIDER_ID = /^[a-z0-9][a-z0-9-_]*$/` plus an exists-and-not-disabled check
  (`dialog-custom-provider-form.ts:69-72`) — so re-adding a disabled provider
  ID is allowed (a "reconnect" path), but stomping a live one is not. The test
  at `dialog-custom-provider-form.test.ts:54-71` pins this exact behavior.
- `^https?:\/\//` on `baseURL` (`:74`) — minimal but correct for the
  openai-compatible adapter.
- Per-model dedup via `seenModels` (`:78-86`) and per-header dedup via
  case-insensitive `normalized = key.toLowerCase()` (`:91-99`). Header dedup is
  the right key — HTTP header names are case-insensitive, so `X-Test` and
  `x-test` should collide.
- Empty `{key: "", value: ""}` rows are skipped (`:93`), so the UI can show an
  always-empty "add another" row without tripping validation.

**`apiKey` resolution (`:102-103`).** `apiKey.match(/^\{env:([^}]+)\}$/)` — if
the user typed `{env: CUSTOM_PROVIDER_KEY}`, the env-var name is hoisted into
`config.env` and `key` is left undefined; otherwise the literal value is
returned in `key`. This matches the WebUI shape (per the PR description) and
keeps secrets out of the on-disk config when an env-ref is given.

**UI (`dialog-provider.tsx:35-50`).** A synthetic `custom` option with
`value: "__custom_provider__"` is prepended to the picker before the
`PROVIDER_PRIORITY` sort. Selecting it routes to `CustomProviderMethod({
dialog, sdk, sync, toast })` (function not in the visible diff window but
referenced from the import at line 18). The pre-PR `pipe(...)` chain is now
spread into `[custom, ...pipe(...)]`, preserving existing per-provider
`onSelect` behavior verbatim.

## Risks / nits

- The `__custom_provider__` sentinel collides with any real provider ID
  prefixed with `__`. The validator's `PROVIDER_ID` regex disallows `_` as a
  leading char (`^[a-z0-9]`), so user-entered IDs cannot collide — but it's
  worth a `// reserved sentinel` comment near line 36 so a future refactor
  doesn't move the regex without noticing.
- `apiKey.match(/^\{env:([^}]+)\}$/)` requires exactly `{env:NAME}` with no
  spaces around the colon — the test feeds `" {env: CUSTOM_PROVIDER_KEY} "`
  and relies on the outer `.trim()` plus the inner `.trim()` on the captured
  group (`:102`). That works, but `{env :NAME}` (space before colon) silently
  falls through and gets stored as a literal key. Two-line fix; not a blocker.
- No test for the `disabledProviders` "reconnect" path showing the previous
  config is replaced rather than merged — the validator returns a fresh
  `config` object, so this is correct, but a one-line assertion would lock it.

## Verdict

**`merge-after-nits`**

Validator is well-shaped (tagged union, deterministic order, all error
strings actionable), tests cover the two highest-value paths (happy and
duplicate-model + reconnect), and the UI integration is the minimal
prepend-to-options change. The `{env:...}` parser is the only place where
behavior could subtly drift from the WebUI flow; tightening the regex to
`/^\{env:\s*([^}]+?)\s*\}$/` would close that gap and could land in the same
PR. The reserved-sentinel comment is worth landing too.

## What I learned

The pattern of returning a single tagged-union `Validation` instead of
throwing is the right shape for a wizard-style TUI: each step renders the
`error` string verbatim, and the success branch carries exactly the data the
next step needs (`providerID`, `name`, `key`, `config`) with no re-derivation.
The `disabledProviders` carve-out in the existing-IDs check is the kind of
thing you only get right if you've seen a user disable a provider, then try
to re-add it with the same ID, and watch the form refuse them.
