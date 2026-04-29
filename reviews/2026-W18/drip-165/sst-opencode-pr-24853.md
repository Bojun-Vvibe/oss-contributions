# sst/opencode #24853 — Effect HttpApi backend parity + observable backend selection

- **PR:** https://github.com/sst/opencode/pull/24853
- **Head SHA:** `5b1feb4c54d6729b3a209adec68fbe24364ffb71`
- **Size:** +4294 / −2770 across ~80 files

## Summary
Lands the Effect HttpApi server backend alongside the existing Hono backend, making backend selection an observable property of every served request. Splits HttpApi composition into per-feature `groups/*` route contracts and `handlers/*` implementations, expands SDK + JSON parity tests across both backends, and tightens schema annotations (`Schema.Number` → `Schema.Finite`, ad-hoc `Schema.Number` → `NonNegativeInt` for `Oauth.expires`).

## Specific observations

1. **Backend selection lifted into a typed module at `packages/opencode/src/server/backend.ts:609-647`** — `Selection = { backend: "effect-httpapi" | "hono", reason: "env" | "stable" | "explicit" }`, with `attributes()` returning the four-key OTel-style attribute bag (`opencode.server.backend`, `opencode.server.backend.reason`, `opencode.installation.channel`, `opencode.installation.version`). Good move: the previous one-liner `Flag.OPENCODE_EXPERIMENTAL_HTTPAPI ? createHttpApi() : createHono({})` left zero forensic trace on which backend served a given request, which was already biting parity-debugging.

2. **`server.ts:7479-7513` wraps both factories in `withBackend()` which logs `"server backend selected"` once per construction**. Per-request attribution comes from `LoggerMiddleware(backendAttributes)` at `middleware.ts:667-688` — which now closes over the backend attribute bag instead of being a bare `MiddlewareHandler`. The refactor preserves the `/log` skip (`if (skip) return next()`) and folds the dual `log.info` + `log.time` calls into one timed span — a small but real correctness fix because the old code emitted `log.time` even for `/log` requests and only conditionally stopped the timer (`if (!skip) timer.stop()`), leaving zombie timers if anything threw between `time` and `stop`.

3. **`Legacy(opts)` export at `server.ts:7502-7504`** explicitly stamps `reason: "explicit"` regardless of the env flag. This is the right escape hatch for tests that need to pin the legacy path (and the new `httpapi-bridge.test.ts` exercises it). Worth checking: any caller that previously imported `create()` and got the env-flag-driven backend will now silently keep getting it — no migration needed, but the `Legacy()` symbol becoming public surface should be documented in the spec at `specs/effect/http-api.md` (the PR adds 8 lines there but they are about V2 cleanup, not about the `Legacy()` opt-out).

4. **HttpApi composition split into `groups/*` + `handlers/*`** (lines 1776, 2278, 2504, 2584, 2809, 3243, 3339, 3509 of the diff add `groups/{global,metadata,provider,pty,session,sync,tui,workspace}.ts`). This is the right shape — route contracts (Effect `HttpApi` schemas) live separately from handler bodies, mirroring how Hono's `routes/instance/*` already factors. The aggregate is composed in `httpapi/server.ts` (line 6604). Net effect: SDK type-gen now has a single canonical input per surface area.

5. **`scripts/diff-sdk-types.sh`** (lines 1-58 of the diff, new file, executable) is the load-bearing parity tool — generates SDK types from both backends, normalizes (sorts `export type|function|const` blocks alphabetically), and diffs. The `--stat` mode reports `Hono-only` vs `HttpApi-only` type counts. This is the right gate for the eventual Hono retirement; it should be wired into CI before the deletion phase, not after.

6. **`workspace.ts:7540-7564`** renames `local` → `isLocalWorkspaceRoute`, `getSessionID` → `getWorkspaceRouteSessionID`, `socket` → `websocketTargetURL`, and adds `workspaceProxyURL(target, requestURL)` that does the URL-rewrite-plus-strip-`workspace`-param dance previously inlined. All four are now exported because the HttpApi handler tree needs the same logic. Naming change is a net positive (`local` was ambiguous), though it's a public-surface rename — not a problem for an internal package, but anything importing from `@opencode-ai/core` should be re-checked.

7. **Schema tightening (`agent/agent.ts:88-99`):** `topP`, `temperature`, `steps` move from `Schema.Number` to `Schema.Finite`. Correct — `Number` accepts `NaN` and `±Infinity`, which serialize as `null` in JSON and would have produced silent provider-side errors. Same pattern in `auth/index.ts:119` switching `Oauth.expires` to `NonNegativeInt` (so a negative `expires` from a malformed OAuth response can't pretend to be a valid future timestamp). Both are behavior-changing on the validation boundary — anyone with stored OAuth state where `expires` was somehow negative or non-integer (rare but possible from mismatched provider parsers) will now fail to load that auth blob and have to re-authenticate. Worth a release-note line.

8. **`bus/bus-event.ts:131-143` adds `effectPayloads()`** that emits `Schema.Struct({ type: Literal(type), properties: def.properties }).annotate({ identifier: "Event.${type}" })` for each registered bus event — the Effect-side analogue of the existing `payloads()` function. Identifier annotation is what lets the HttpApi OpenAPI generator emit named component schemas instead of inline structs. Good.

9. **`Default()` and `create()` now both pay one `select()` call** to decide the backend. Cheap (it's just a flag read), but `select()` is recomputed on every `create()` call — if a server is rebuilt mid-process and the env flag flipped (it can't, in practice, but the type allows it), you can get backend-divergent server instances. Not a real bug today, just an observation.

## Risks

- **Public-surface rename** in `workspace.ts` — any out-of-tree consumer of `@opencode-ai/core` calling `local()` or `getSessionID()` from that module breaks. Likely zero such consumers but worth verifying with `rg -tts "from.*server/workspace"` across the workspace.
- **`Schema.Finite` / `NonNegativeInt`** is a stricter validation contract — any in-the-wild persisted state with non-finite `topP`/`temperature`/`steps` or negative `Oauth.expires` will start failing to load. Most likely zero impact, but a one-line release note ("strict numeric validation on persisted agent + OAuth state") is cheap insurance.
- **HttpApi backend is still env-gated** (`Flag.OPENCODE_EXPERIMENTAL_HTTPAPI`), so the default user path is unchanged. Real risk surface = whoever opts in — and the parity tests look thorough (the PR description lists 7 distinct `httpapi-*.test.ts` files run in CI).

## Suggestions

- **(Recommended)** Wire `scripts/diff-sdk-types.sh --stat` into CI as a non-blocking advisory job today, blocking once the deletion phase begins. Otherwise the parity drift between backends is invisible until a release lands.
- **(Recommended)** Document `Legacy()` in `specs/effect/http-api.md` — it's now a stable test seam and should be listed alongside the V2 cleanup notes.
- **(Optional)** Cache `select()` once at module load if there's never a legitimate reason to re-evaluate the env flag mid-process. Today it's recomputed per `create()`/`Default()` call.
- **(Nit)** The `withBackend()` log line `"server backend selected"` fires once per construction but the same `Selection` is logged via `LoggerMiddleware` on every request — slight redundancy, but cheap and useful for forensic correlation.

## Verdict: `merge-after-nits`

Architecturally the right move. Backend selection becoming a first-class typed value with attribute fan-out is exactly the observability needed before any meaningful deletion of the Hono path. Parity surface (groups/handlers split, SDK diff script, expanded test files) is thorough. The two real concerns — public-surface rename in `workspace.ts` and the strictness change on `Oauth.expires` — are documentation-level, not code-level.

## What I learned

The pattern of "lift a binary mode toggle into a typed `Selection { mode, reason }` value, then thread its attribute bag through the request log" is a clean way to retire feature flags without losing the forensic trail. The `reason` field doing double duty as both audit (`"env"` vs `"stable"` vs `"explicit"`) and selection diagnosis is what makes it useful — a bare `backend: "hono" | "effect-httpapi"` would tell you what but not why.
