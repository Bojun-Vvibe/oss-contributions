# openai/codex PR #20002 — tighten network proxy bypass defaults

- URL: https://github.com/openai/codex/pull/20002
- Head SHA: 31acacfe1846
- Files: `codex-rs/network-proxy/src/proxy.rs`
- Verdict: **merge-as-is**

## Context

`DEFAULT_NO_PROXY_VALUE` is the bypass list seeded into the
sandbox's `NO_PROXY` env var. The list previously included the
`169.254.0.0/16` link-local block alongside the loopback and
RFC 1918 ranges. The PR removes the `169.254.0.0/16` entry. The
single test that asserted its presence is flipped from
`assert!(no_proxy.contains(...))` to `assert!(!no_proxy.contains(...))`.

## Design analysis

The diff is two lines plus the test flip. The reasoning behind
removing `169.254.0.0/16` from the bypass list is straightforward
once you remember what's actually at link-local addresses on cloud
hosts: the instance metadata service (`169.254.169.254` on EC2,
GCE, Azure, etc.). Bypassing the proxy for that range means
sandboxed processes can reach the metadata endpoint directly,
side-stepping any proxy-side allow/deny policy and any audit logs.
That's a textbook SSRF/credential-exfiltration vector.

The remaining bypass entries (`localhost`, `127.0.0.1`, `::1`,
RFC 1918) all describe traffic that genuinely shouldn't transit
the proxy in any reasonable deployment: loopback is loopback, and
RFC 1918 addresses on a sandboxed process are typically the
container's own internal services. Link-local is the odd one out
because (a) it's globally well-known to host the metadata service
and (b) sandboxed code has no legitimate reason to reach it.

## Risks / suggestions

1. **Behavior change for users who relied on link-local
   reachability.** If anyone is, e.g., running an mDNS responder
   on a link-local address and expected the sandbox to talk to
   it directly, they now have to add it back via
   `NO_PROXY` config. Worth a one-liner in release notes.
2. The test message at line 1011 should arguably say "and the
   link-local block must NOT be bypassed by default to keep cloud
   metadata endpoints behind the proxy" — pure documentation, but
   it would prevent a future contributor from "fixing" the test
   by re-adding the entry.
3. Consider whether `0.0.0.0/8` and IPv6 ULA (`fc00::/7`) deserve
   the same treatment as a follow-up. Out of scope here, just a
   thought.

## What I learned

Default deny-lists for proxy bypass are security boundaries
disguised as performance optimizations. Every entry in such a
list is "we trust traffic to this range to leave the proxy's
visibility" — and link-local on cloud hosts is exactly the range
you do *not* want to grant that trust to. This is a one-line PR
that closes a real attack surface; the test flip alone tells you
the change is intentional and tightly scoped.
