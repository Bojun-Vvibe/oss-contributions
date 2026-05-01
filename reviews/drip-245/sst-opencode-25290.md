# PR #25290 — fix(mdns): derive service name from domain to avoid collisions

- Repo: sst/opencode
- Head: `63e26d70b65b88f7f21cba7f6f246c2d7ce54a9f`
- URL: https://github.com/sst/opencode/pull/25290
- Verdict: **merge-as-is**

## What lands

Fixes #22354. Two opencode instances sharing a port on the same LAN both
advertised `opencode-<port>` as their mDNS service instance name even when
they had different `--mdns-domain` values, so the second instance failed
with `Service name is already in use on the network`. The fix derives the
service-name prefix from the configured domain.

## Specific findings

- `packages/opencode/src/server/mdns.ts:14-19`:
  ```
  export function serviceName(port: number, domain?: string): string {
    const host = (domain ?? "opencode.local").trim()
    const stripped = host.replace(/\.local\.?$/i, "").replace(/\./g, "-")
    const prefix = stripped || "opencode"
    return `${prefix}-${port}`
  }
  ```
  The `\.local\.?$/i` regex correctly handles both `vmb.local` and
  `vmb.local.` (FQDN trailing dot), the `replace(/\./g, "-")` turns
  multi-label customs (`team.dev.local` → `team-dev`) into valid DNS-SD
  service-name prefix characters, and the `stripped || "opencode"` fallback
  ensures `.local` alone (which strips to empty) doesn't produce
  `-<port>` (which would be both ugly and Bonjour-rejected).
- Backwards compatibility check: with default `opencode.local`, the regex
  strips to `"opencode"` so the resulting name is `opencode-<port>` — the
  exact pre-PR value. Single-instance users see no behavior change.
- `mdns.ts:25-26` switches `publish()` to call the new helper. The
  `currentPort === port` short-circuit at the top of `publish()` is
  unchanged; that's correct because the bug only manifests across
  *different* processes on the same LAN, not within a single process.
- Test coverage at `test/server/mdns.test.ts:1-40` is thorough: 9 test
  cases exercise default domain, custom domain, trailing-dot FQDN,
  empty-prefix fallback, multi-label, whitespace trim, and the explicit
  collision case from the bug report (line 22-24 asserts two distinct
  domains on the same port produce distinct names — exactly the
  regression to lock in).

## Verdict rationale: merge-as-is

- Pure helper, fully tested, default behavior preserved.
- The regex is anchored (`$/i`) so it can't accidentally strip an inner
  `.local` segment.
- One small thing worth noting: DNS-SD instance names allow many more
  characters than `[a-zA-Z0-9-]`, so theoretically `vmb_dev.local` would
  produce `vmb_dev-4096` which is fine. The `replace(/\./g, "-")` is just
  enough to keep the result a valid single label, not over-eager
  sanitization. Good restraint.
