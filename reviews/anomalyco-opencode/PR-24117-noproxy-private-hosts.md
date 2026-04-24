# PR-24117 — fix(provider): allow remote local-network hosts when proxy env vars are set

[sst/opencode#24117](https://github.com/sst/opencode/pull/24117)

## Context

Adds `ensureNoProxyForBaseURL(baseURL)` in
`packages/opencode/src/provider/provider.ts`. When the user has
`HTTP_PROXY` / `HTTPS_PROXY` exported (a common corporate setup) and
configures a provider with a `baseURL` pointing at a local-network host
(e.g. `http://192.168.0.100:1234/v1` for a LAN-hosted Ollama / vLLM /
LM Studio), the request was being routed through the corporate proxy,
which then either rejected it or couldn't reach the LAN. The fix
detects private/loopback/link-local hostnames in the `baseURL` and
appends them to both `NO_PROXY` and `no_proxy` if not already covered.

## Why it matters

This is the "I configured a local model and nothing works on my work
laptop" foot-gun. Silent proxy interception is the worst class of
network bug because the failure surfaces as a generic provider timeout
or 502, far from the actual cause.

## Strengths

- The classification helpers (`privateIPv4`, `privateIPv6`,
  `bypassProxy`) are tight and readable: RFC1918 ranges (`10/8`,
  `192.168/16`, `172.16-31/12`), loopback (`127/8`, `::1`), link-local
  (`169.254/16`, `fe8x`–`febx`), and ULA (`fc00::/7` via `fc`/`fd`
  prefix). Coverage is correct.
- Hostname-only single-label rule (`!hostname.includes(".")`) catches
  bare LAN names like `nas`, `printer`, `ollama-box`. Plus `.local`
  catches mDNS.
- Idempotent: parses existing `NO_PROXY`, splits/trims, only appends if
  no entry already covers (with `.suffix` wildcard support via
  `coveredByNoProxy`). Won't grow `NO_PROXY` unboundedly across multiple
  provider creations.
- Only fires when at least one of the proxy env vars is set
  (`proxyKeys.some(...)`). On hosts without a proxy this is a no-op.
- Test pair covers both directions: private host *gets* added, public
  host (`api.openai.com`) *doesn't*. Test save/restores env vars to
  avoid leaking state to sibling tests.

## Concerns / risks

- **Mutating `process.env` is a global side effect.** Once
  `ensureNoProxyForBaseURL` runs, every subsequent fetch in the process
  — including by unrelated SDKs that read `NO_PROXY` lazily — sees the
  appended host. If the user has *two* providers, one local and one via
  a deliberately-routed local proxy, this could surprise them. A
  per-request agent/dispatcher would be safer but a much bigger refactor.
- The fix runs inside the provider layer's `if (baseURL !== undefined)`
  block at line ~1490. If `baseURL` is set later (post-construction) by
  a plugin or a runtime override, this won't re-fire.
- `privateIPv6` is conservative and misses some valid forms — e.g.
  `0:0:0:0:0:0:0:1` (uncompressed loopback), or any IPv6 written with
  uppercase letters not on the `host.toLowerCase()` path (it does
  lowercase, but the `host.startsWith("fe8")` style is fragile vs
  zero-padded forms like `fe80:0000:...`). A `net.isIP` + numeric range
  check would be more robust.
- `coveredByNoProxy` only matches `*`, exact, or `.suffix` patterns.
  Real-world `NO_PROXY` entries also include CIDR notation (used by
  curl, Go's `httpproxy` package, etc.); those won't match an existing
  `192.168.0.0/16` entry and the host will get appended redundantly
  (harmless but ugly).
- No test for the "private + already in NO_PROXY" no-op path, and no
  test for the `URL` parse failure (`new URL("not a url")`) early
  return.

## Suggested follow-ups

- Use `node:net.isIP()` to narrow to actual IPs before doing the
  prefix-string IPv6 checks; drop the prefix heuristics in favor of
  numeric range checks against the parsed octets/groups.
- Add a CIDR-aware short-circuit in `coveredByNoProxy` so existing
  `NO_PROXY=192.168.0.0/16` entries cover any LAN host.
- Document the `process.env` mutation in the function's JSDoc — explicit
  "this is intentional and global" is much friendlier than future
  readers wondering if it's a bug.
- Optional: emit a one-line `log.info` when the function actually
  appends a host, so users debugging "why is my LAN model still being
  proxied" get a hit in the logs.
