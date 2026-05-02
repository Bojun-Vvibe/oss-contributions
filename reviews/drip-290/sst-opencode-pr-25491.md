# sst/opencode PR #25491 — feat(lsp): add languageId field to custom LSP server config

- Head SHA: `3e1a34636c342ee5ba7412a444594748dc526395`
- URL: https://github.com/sst/opencode/pull/25491
- Size: +97 / -2, 6 files
- Verdict: **merge-after-nits**

## What changes

Adds an optional `languageId` field to custom LSP server config
(`packages/opencode/src/config/lsp.ts:17`) so users with bridge servers
(e.g. CFML → coldfusion, custom QML, etc.) can override the default
language ID derived from `LANGUAGE_EXTENSIONS[extension]`. The override
is plumbed through `LSPServer.Info` (server.ts:61), passed into
`LSPClient.create(...)` (lsp.ts:252), and consulted at the
`textDocument/didOpen` payload site (client.ts:597) so the value
actually reaches the LSP server.

Two new tests (test/lsp/client.test.ts:147-181) exercise both the
default path (`.ts` → `typescript`) and the override path
(`.qml` + `languageId: "qml"`). The fake LSP server gains a
`test/get-last-did-open` request method (fake-lsp-server.js:211-214)
to read back the last `didOpen` payload, which is a clean way to
assert this without hitting a real LSP.

## What looks good

- The fallback chain `input.languageId ?? LANGUAGE_EXTENSIONS[extension]
  ?? "plaintext"` (client.ts:597) preserves existing behavior for every
  user that doesn't set the new field. Strict additive change.
- Test for the default-path (`didOpen uses LANGUAGE_EXTENSIONS when no
  languageId override`, test/lsp/client.test.ts:147-155) is a real
  regression guard, not just a happy-path for the new feature.
- Schema test (test/config/lsp.test.ts:54-60) uses a CFML example
  (`coldfusion` languageId), which is a realistic motivating use case
  and not a contrived stub.

## Nits

1. `lsp.ts:187` carries the override forward via
   `item.languageId ?? existing?.languageId`. Worth a short comment that
   user config wins over baked-in defaults — anyone overriding a
   built-in server (e.g. supplying their own typescript-language-server)
   should know that setting `languageId: "vue"` will silently flip a
   built-in entry.
2. There's no validation that `languageId` is a non-empty string at the
   schema layer (`lsp.ts:17`). Empty string would fall through the
   `??` chain (since `??` only catches null/undefined) and result in
   `didOpen` being sent with `languageId: ""`, which most LSPs reject.
   Either reject empty in the schema (`Schema.String.pipe(
   Schema.minLength(1))`) or coerce empty → undefined in the consumer.
3. The test fake-lsp-server.js:211 stores `lastDidOpen` as a singleton.
   If a future test opens two files in one server lifetime and asserts
   on the first one, it'll silently get the second. Not a bug today,
   but consider an array, especially since the same fake server backs
   multiple tests.

## Risk

Low. Pure additive change with the right fallback semantics, decent
test coverage, and the underlying LSP behavior is well-defined
(`textDocument/didOpen.languageId` is part of the spec). Worst case,
a user sets `languageId: "frobnitz"` and their server complains —
the failure is local to that user.
