---
pr: 3649
repo: QwenLM/qwen-code
title: "fix(lsp): expose status and startup diagnostics"
url: https://github.com/QwenLM/qwen-code/pull/3649
head_sha: 92ed49deb3003520c0335526a3740736b5277d36
author: yiliang114
verdict: merge-after-nits
date: 2026-04-27
---

# QwenLM/qwen-code #3649 — expose LSP status and startup diagnostics

Draft follow-up to #3615 (now retargeted to `main` directly so it
doesn't depend on #3615 merging first). Adds operator-visible LSP
status surfacing: a `getStatusSnapshot()` API on the LSP service, debug
logging at discovery + startup, and a one-liner LSP summary in the
`/status` (`/about`) output. 19 files changed, +727/-14.

## What the diff actually does

### Service plumbing

`packages/cli/src/config/config.ts:1259-1280` — wraps the existing
`lspService.discoverAndPrepare()` and `lspService.start()` calls with
debug-mode-only `getStatusSnapshot()` debug logs. Also captures startup
errors into `config.setLspInitializationError(err)` so the snapshot
reports them rather than just warning to the console.

```ts
await lspService.discoverAndPrepare();
if (config.getDebugMode()) {
  debugLogger.debug('Native LSP status after discovery:', lspService.getStatusSnapshot());
}
await lspService.start();
if (config.getDebugMode()) {
  debugLogger.debug('Native LSP status after startup:', lspService.getStatusSnapshot());
}
```

### `/status` integration

`packages/cli/src/utils/systemInfo.ts:198-238` — new `getLspStatus(context)`
+ `formatLspStatusSnapshot(snapshot)` helpers. Output strings:
- `disabled` (LSP off)
- `enabled, initialization failed: <err>`
- `enabled, status unavailable` (snapshot.statusUnavailable)
- `enabled, no servers configured` (configuredServers === 0)
- `enabled, N/M ready` plus optional ` (X failed, Y starting, Z not started)`

`packages/cli/src/ui/commands/aboutCommand.ts:34-40` plumbs `lspStatus`
into the about message. `systemInfoFields.ts:32` adds the LSP field
between IDE Client and OS.

### Tests

`packages/cli/src/config/config.test.ts:725-748` asserts
`getStatusSnapshot` is called *twice* during debug startup (after
discovery + after start) and *zero* times during normal startup.
`systemInfo.test.ts:336-396` exercises both the
"enabled, 1/2 ready (1 failed)" path and the "status unavailable" path.
`aboutCommand.test.ts:343-374` asserts the `/about` message contains
`LSP: enabled, 1/1 ready`.

### Docs

`docs/users/features/lsp.md:340-475` — substantive rewrite of
"Server Not Starting" and "Debugging" sections. Promotes
`--experimental-lsp` to step 1, drops the obsolete `DEBUG=lsp*` env
incantation in favor of `qwen --experimental-lsp --debug`, adds the
specific log-prefix list (`[LSP]`, `[CONFIG] Native LSP status after
discovery`, `[STATUS] LSP status snapshot for /status`), enumerates
the four expected `/status` output strings, and lists common error
messages (`command path is unsafe`, `command not found`,
`requires trusted workspace`, `LSP connection closed`) with one-line
fixes for each.

## Observations

1. **`/lsp status` slash command is gone, replaced by `/status`.** The
   docs change at line 462 says
   > Use the `/lsp status` command to see all configured and running language servers.
   ↓ replaced with
   > Start Qwen Code with LSP and debug mode enabled… Then run `/status`.
   This is a UX regression for users who knew `/lsp status` — it goes
   from a one-line dedicated command to "you must launch with `--debug`
   then run `/status`". Two issues:
   - the `--debug` requirement is for the *log inspection* part, not
     the `/status` output; `/status` shows the LSP summary regardless
     of `--debug` (verified at `aboutCommand.test.ts:367`, no debug mode
     mocked). The docs should say so explicitly.
   - if `/lsp status` was previously a separate slash command, deleting
     it without a deprecation notice will break user muscle memory.
     Either leave `/lsp status` as an alias for the LSP-specific section
     of `/status`, or add a one-line "deprecated, use `/status` instead"
     in the slash command itself.

2. **`formatLspStatusSnapshot` ordering** at `systemInfo.ts:212-238`:
   `failed → starting → not started`. That's the right priority for an
   operator scanning the line, but consider also surfacing
   `inProgressServers > 0` more loudly when `configuredServers ===
   inProgressServers && readyServers === 0`. Currently
   `enabled, 0/2 ready (2 starting)` reads correctly but a user might
   want `(starting…)` or a spinner glyph to know to wait vs. file a bug.
   Optional polish.

3. **Error-string interpolation safety.** Line 219:
   ```ts
   return `enabled, initialization failed: ${snapshot.initializationError}`;
   ```
   `initializationError` could be `Error` or string (config.ts:1283 sets
   `err instanceof Error ? err : String(err)` — so it's `Error | string`).
   When it's an `Error`, JS coerces to `${err}` which calls
   `Error.prototype.toString()` returning `"Error: msg"`. Fine for an
   Error with a message; ugly when the underlying error has a stack
   that toString picks up. Consider:
   ```ts
   const errStr = snapshot.initializationError instanceof Error
     ? snapshot.initializationError.message
     : String(snapshot.initializationError);
   ```
   Cosmetic; doesn't block.

4. **Snapshot is called twice unconditionally during debug startup** at
   `config.ts:1262` and `:1269`. If `getStatusSnapshot()` is expensive
   (does it walk all servers and stringify?) this doubles the cost
   in debug. Glance at the implementation in core; if it's a simple
   accessor, ignore. If it constructs the snapshot from live data, gate
   one of the two calls behind a "diff vs. previous snapshot" check.

5. **Test in aboutCommand.test.ts uses a hand-rolled mock** of
   `getExtendedSystemInfo` rather than going through the real path.
   That's fine for unit-testing the rendering, but means no integration
   test actually verifies that `Config.getLspStatusSnapshot()` →
   `getExtendedSystemInfo().lspStatus` → `aboutCommand` rendering. The
   chain is so simple a single end-to-end test would lock it.

6. **`addField(fields, 'LSP', info.lspStatus ?? '')`** at
   `systemInfoFields.ts:32` always adds the LSP row, with empty value
   when LSP isn't configured. That changes `/about` output for users
   who don't use `--experimental-lsp`: they now see an empty `LSP:`
   line. Either guard with `if (info.lspStatus) addField(...)`, or
   render `'disabled'` when LSP isn't enabled (which is what the
   formatter already does — but only if `getStatusSnapshot()` is
   called, which it isn't when no LSP service exists). Worth a check
   against the actual behavior on a fresh project with no
   `--experimental-lsp`.

7. **Docs mention `~/.qwen/debug/latest`** at lsp.md:381 but `latest`
   is a symlink in some installs and a literal directory in others.
   Add `# (a symlink to the latest session log directory)` so users
   know what to expect.

## Verdict

**merge-after-nits.** Solid debuggability improvement with reasonable
test coverage at the unit layer. Three concrete asks:

1. Confirm/document the `/status` LSP line works *without* `--debug` and
   keep `/lsp status` as an alias (obs 1).
2. Skip rendering the LSP field on `/about` when LSP is fully disabled
   (obs 6) so output stays clean for non-LSP users.
3. Tighten `initializationError` formatting to use `.message` for Error
   instances (obs 3).

Each is 1-3 lines.

## Banned-string scan

None. (Note: docs refer to LSP but the canonical "language server
protocol" is unambiguous and not a banned term.)
