# QwenLM/qwen-code#3635 — feat(core): --insecure flag and QWEN_TLS_INSECURE env var (#3535)

- **URL**: https://github.com/QwenLM/qwen-code/pull/3635
- **Head SHA**: `b1eb211a126a`
- **Diffstat**: +321 / -20 (14 files)
- **Verdict**: `request-changes`

## Summary

Adds an opt-in TLS-verification skip path for outbound HTTPS to model APIs and MCP servers, motivated by Node `fetch` (undici) ignoring `NODE_TLS_REJECT_UNAUTHORIZED`. Introduces three coordinated entry points with explicit precedence: `--insecure` CLI flag > `QWEN_TLS_INSECURE=1|true|yes` > `NODE_TLS_REJECT_UNAUTHORIZED=0`. When set, threads `connect: { rejectUnauthorized: false }` into the OpenAI SDK undici dispatcher (default + DashScope), the Anthropic SDK dispatcher, and the **global** undici dispatcher used by MCP / streaming / telemetry. Closes #3535.

## Findings

### Security model
- This is a footgun by design — disables TLS verification for *all* outbound traffic in the process when set, including telemetry. The PR is honest about this in the "Areas needing careful review" section, but the reviewer concern is whether a bare `--insecure` (no scope) is the right surface. Compare to curl, which has `--insecure` apply only to the current request, not globally. A safer surface would be `--insecure-host <hostname>` (allowlist) so the user can opt into "I know this one self-signed dev box is fine" without dropping verification for telemetry to a Qwen-controlled endpoint.
- `packages/core/src/utils/runtimeFetchOptions.ts:+45 / -6` — the global-dispatcher mutation (`setGlobalDispatcher(new Agent({ connect: { rejectUnauthorized: false } }))`) is process-wide and irreversible within the process lifetime. PR body acknowledges this matches the existing proxy-path pattern. That precedent is itself questionable; replicating it here doubles the blast radius.

### Behavior precedence
- `packages/cli/src/config/config.ts:+44 / -0` and `config.test.ts:+95 / -0` — 7 new test cases pinning the `--insecure` > `QWEN_TLS_INSECURE` > `NODE_TLS_REJECT_UNAUTHORIZED` precedence, including the "`--insecure` overrides `QWEN_TLS_INSECURE=0`" interaction. Coverage looks thorough.
- Honoring `NODE_TLS_REJECT_UNAUTHORIZED=0` is a real backwards-compat concession (Node `fetch` itself doesn't), and matches Claude Code behavior. Reasonable. But this means a user who exports `NODE_TLS_REJECT_UNAUTHORIZED=0` for a *different* tool will silently disable TLS verification in qwen-code too. At minimum, a one-line stderr warning on startup ("TLS verification is disabled because NODE_TLS_REJECT_UNAUTHORIZED=0 is set in your environment") would prevent users from being surprised.

### API shape change
- `packages/core/src/utils/runtimeFetchOptions.ts` — `buildRuntimeFetchOptions` second arg now accepts either a bare proxy-URL string (legacy) or a `RuntimeFetchConfig` object. Pinned by `'treats a bare proxy-URL string identically to legacy callers'` regression test. Backward-compatible, fine.

### Surface and naming
- PR author offers to rename to `tlsRejectUnauthorized: false` to match Node's exact spelling. Strongly support that — `insecure` is colloquial; matching Node's spelling makes the option self-documenting and grep-able for security review.
- `packages/cli/src/commands/auth/handler.ts:+1 / -0` and `gemini.test.tsx:+1 / -0` — `insecure: undefined` literal additions to satisfy `CliArgs` shape. Stylistically fine but suggests `insecure` should arguably be optional in the `CliArgs` interface. Minor.

### Tests
- `runtimeFetchOptions.test.ts:+77 / -0` — 5 new cases pinning `connect` option behavior on `Agent` and `ProxyAgent`, plus the string-vs-object equivalence. Good.
- Provider tests (`default.test.ts`, `dashscope.test.ts`, `anthropicContentGenerator.test.ts`) updated for `getInsecure()`. Mechanical.
- **Missing**: a smoke test that demonstrates an actual self-signed HTTPS server is reachable with `--insecure` and unreachable without — the unit tests verify the option is *plumbed*, not that the network behavior actually changes. Important for a security-sensitive feature.

## Recommendation

`request-changes`. The user need is real and the implementation is competent and well-tested at the option-plumbing layer. Two things should change before merge:
1. **Stderr warning** when TLS verification is disabled at startup (regardless of which of the three layers triggered it) — users should never be silently insecure.
2. **Rename** to `tlsRejectUnauthorized` per the author's own offer, for grep-ability and parity with Node.
Strongly consider also adding `--insecure-host <hostname>` as a narrower default and treating the global `--insecure` as a power-user escape hatch.
