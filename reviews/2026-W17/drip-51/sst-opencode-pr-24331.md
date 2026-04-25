---
pr: 24331
repo: sst/opencode
sha: 8a85782135ef3720e25a13b84f3fab0bd72883cd
verdict: merge-as-is
date: 2026-04-26
---

# sst/opencode#24331 — fix: add logging to silent catch blocks across core modules

- **URL**: https://github.com/sst/opencode/pull/24331
- **Author**: alfredocristofano

## Summary

Mechanical sweep replacing `} catch {}` and `} catch (err) {}` with
`} catch (e) { log.debug/warn(...) }` across 9 modules:
`auth/index.ts`, `global/index.ts`, `mcp/index.ts`,
`provider/error.ts`, `pty/index.ts`, `server/mdns.ts`,
`session/llm.ts`, `session/message-v2.ts`, `session/session.ts`,
`util/error.ts`, `util/filesystem.ts`. Adds a per-file
`log = Log.create({ service: "..." })` where missing.

Also includes one non-logging change at the bottom of `session.ts`:
`export * as Session from "./session"` (self-export pattern).

## Reviewable points

- Severity choices look right:
  - `warn` for things a user/operator should notice:
    - `auth/index.ts:65` — `OPENCODE_AUTH_CONTENT` parse failure
      (config bug, env was set but unparseable)
    - `global/index.ts:56` — cache cleanup failure
    - `server/mdns.ts:39` — bonjour destroy failure (port
      binding may be sticky next start)
    - `session/session.ts:509` — session remove failure (data
      consistency)
  - `debug` for expected/recoverable noise:
    - `mcp/index.ts:524` — SIGTERM to gone process
    - `provider/error.ts:74` — non-JSON response body
    - `pty/index.ts:125-130` — process.kill / ws.close on
      teardown
    - `util/filesystem.ts:20`, `:122`, `:239` — stat/realpath
      failures during scans

- `session/llm.ts:258` — bonus correctness fix bundled in:
  `catch (e: any)` → `catch (e: unknown)` with proper
  `e instanceof Error ? e.message : String(e)` narrowing. Good
  TypeScript hygiene.

- `session/session.ts:509-510` — note the new code does
  `log.warn(...) ; log.error(e)`. Two log lines per failure,
  same severity-pair pattern as elsewhere in the file. Slightly
  noisy but consistent.

- The `export * as Session from "./session"` line at end of
  `session.ts` is an unrelated barrel-style re-export. Doesn't
  break anything but it's not advertised in the PR title and
  belongs in a separate refactor PR (also note PR #24333 is
  *removing* barrels in this exact area — these two will
  conflict on merge order).

## Rationale

Pure observability win. The "silent catch" pattern is exactly
what masks real production bugs (a corrupted cache cleanup that
silently swallows ENOSPC, an mdns publish that quietly fails so
LAN discovery breaks for one user). Severity choices match the
likely incident class. The bundled `Session` re-export is the
only nit; not blocking.

## What I learned

Severity discipline matters when bulk-converting silent catches:
`debug` for expected-and-recoverable, `warn` for "user can act on
this," `error` for "something is broken and we proceeded anyway."
This PR mostly gets it right, which is what makes the change
actually useful — a sweep that promotes everything to `error`
would just become noise to filter out.
