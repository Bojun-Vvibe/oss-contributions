---
pr: 3635
repo: QwenLM/qwen-code
sha: b1eb211a126afb611abd43e57eb8da98657fb425
verdict: merge-after-nits
date: 2026-04-26
---

# QwenLM/qwen-code #3635 — feat(core): --insecure flag and QWEN_TLS_INSECURE env var (#3535)

- **Author**: JahanzaibTayyab
- **Head SHA**: b1eb211a126afb611abd43e57eb8da98657fb425
- **Size**: +321/-20 across 14 files
- Closes: #3535

## Scope

Adds an escape hatch for self-signed TLS certificates on outbound HTTPS connections (model APIs, MCP servers, telemetry). The right-shaped problem: `NODE_TLS_REJECT_UNAUTHORIZED=0` works for Node's legacy `http` module but is *ignored by undici* (the engine behind `fetch`), so today a user with a self-signed model endpoint sees only `[API Error: Connection error. (cause: fetch failed)]` and has no escape hatch.

This PR adds three coordinated entry points (resolution order, highest to lowest):
1. `--insecure` CLI flag (and `--no-insecure` for explicit override).
2. `QWEN_TLS_INSECURE=1|true|yes` (case-insensitive).
3. `NODE_TLS_REJECT_UNAUTHORIZED=0` — for parity with Node convention and Claude Code.

When set, threads `connect: { rejectUnauthorized: false }` into:
- the OpenAI SDK's undici dispatcher (default + DashScope providers, `provider/default.ts:51-58`, `provider/dashscope.ts:63-70`),
- the Anthropic SDK's undici dispatcher (`anthropicContentGenerator.ts:66-73`),
- the **global** undici dispatcher used by MCP / streaming / telemetry (`packages/core/src/config/config.ts:859-880`).

## Specific findings

- **Resolution order is exactly right** (`packages/cli/src/config/config.ts:151-177`):
  ```ts
  function resolveInsecureFlag(cliFlag: boolean | undefined): boolean {
    if (cliFlag !== undefined) {
      return cliFlag;
    }
    const qwenEnv = process.env['QWEN_TLS_INSECURE']?.trim().toLowerCase();
    if (qwenEnv === '1' || qwenEnv === 'true' || qwenEnv === 'yes') {
      return true;
    }
    if (process.env['NODE_TLS_REJECT_UNAUTHORIZED']?.trim() === '0') {
      return true;
    }
    return false;
  }
  ```
  Three things done right: (1) tri-state CLI flag preserves `--no-insecure` as the highest-precedence "force off" — verified by tests at `config.test.ts:870-895` (`--no-insecure` overrides both env vars). (2) `QWEN_TLS_INSECURE` accepts the three idiomatic truthy values, not just `"1"`. (3) `NODE_TLS_REJECT_UNAUTHORIZED=0` is honored even though `fetch` itself doesn't, because users reasonably expect "the global Node convention applies here" — and the alternative (silent no-op) is the bug.

- **Three-state CLI flag is the subtle right answer.** The yargs definition at `config.ts:303-318` deliberately sets no default, so yargs reports `undefined` when neither flag is passed. That preserves the three states (true / false / undefined) needed for the resolver. Block comment at the option site explains why: "preserves three distinct states (true / false / undefined), which lets the resolver below treat an explicit `--no-insecure` as the highest-precedence override." This is the kind of comment that prevents a future maintainer from "cleaning up" the missing default and silently breaking the override path.

- **Global dispatcher carve-out covers MCP/telemetry transparently** (`packages/core/src/config/config.ts:859-880`):
  ```ts
  const connect = this.insecure ? { rejectUnauthorized: false } : undefined;
  if (proxyUrl) {
    setGlobalDispatcher(new ProxyAgent({ uri: proxyUrl, ...(connect ? { connect } : {}) }));
  } else if (this.insecure) {
    setGlobalDispatcher(new Agent({ connect: { rejectUnauthorized: false } }));
  }
  ```
  The proxy + insecure interaction is handled correctly: if both are set, `ProxyAgent` gets the `connect` option and the insecure setting takes effect *through* the proxy. If only insecure is set with no proxy, a plain `Agent` is installed globally. The branch where neither is set leaves the default global dispatcher untouched — i.e., zero overhead for the 99% of users who don't need this. Verified by the test at `runtimeFetchOptions.test.ts:413-421` ("omits connect option when insecure is unset").

- **Backward-compat shim for `buildRuntimeFetchOptions`** (`packages/core/src/utils/runtimeFetchOptions.ts:516-575`): the old positional `proxyUrl` string call sites still work — `normalizeConfig` accepts `string | RuntimeFetchConfig | undefined` and produces a normalized config. The test at `runtimeFetchOptions.test.ts:455-474` explicitly asserts `buildRuntimeFetchOptions('openai', 'http://proxy.local')` and `buildRuntimeFetchOptions('openai', { proxyUrl: 'http://proxy.local' })` produce identical dispatchers. That's the right kind of test — pin the legacy contract so a future cleanup doesn't accidentally break callers I haven't audited.

- **Test matrix is comprehensive** (`config.test.ts:803-895`): 9 tests covering default-off, CLI-on, env-on (each variant), explicit-zero env-vars (rejected), CLI override of env, and `--no-insecure` override of env. Plus `runtimeFetchOptions.test.ts:411-487`: dispatcher shape with insecure on/off, proxy + insecure combination, Anthropic dispatcher path, legacy-shim equivalence. This is the test density that prevents future regressions.

- **Docs land in the right place** (`docs/users/support/troubleshooting.md:30-33`). The new troubleshooting entry describes the *symptom* the user sees (`[API Error: Connection error. (cause: fetch failed)]`), explains *why* `NODE_TLS_REJECT_UNAUTHORIZED=0` alone doesn't work for fetch-based code, and lists the three escape hatches. Plus the explicit "Only use this on networks you trust" warning. Good failure-mode-first documentation.

- **Nit: missing audit log when insecure is active.** This is a security-relevant flag — disabling cert verification opens MITM exposure. At minimum, log a `verbose_proxy_logger.warning` at startup when `this.insecure === true`, naming which entry point activated it ("--insecure flag", "QWEN_TLS_INSECURE=1", or "NODE_TLS_REJECT_UNAUTHORIZED=0"). Otherwise a stale env var in someone's CI shell silently disables TLS verification for every Qwen Code run, and the user has no breadcrumb when investigating a later compromise.

- **Nit: no per-host carve-out.** This is "all-or-nothing" — once enabled, every outbound HTTPS connection (model API, MCP, telemetry, future web-fetch tools) skips verification. The Claude Code precedent is the same, so this is acceptable for v1, but document the limitation: a user who wants insecure access *only* for `https://homelab.local:8443` and *strict* verification everywhere else has no option here. A `QWEN_TLS_INSECURE_HOSTS=homelab.local,*.dev.lan` allowlist would be the right v2 enhancement; flag it in a follow-up issue.

- **Nit: `QWEN_TLS_INSECURE` accepts `"true"` / `"yes"` / `"1"` but `NODE_TLS_REJECT_UNAUTHORIZED` only checks `=== "0"`.** The Node convention is documented as `"0"`, so this is correct, but a user might try `NODE_TLS_REJECT_UNAUTHORIZED=false` and silently get strict verification. Add one comment line above the strict-equality check: `// Node convention: only the literal string "0" disables verification; "false"/"no" are silently ignored upstream too.`

- **Nit: `--insecure` should ideally print a one-line stderr banner on startup** (e.g., `WARNING: TLS certificate verification is disabled (--insecure / QWEN_TLS_INSECURE / NODE_TLS_REJECT_UNAUTHORIZED=0). MITM exposure on untrusted networks.`). Curl, openssl, and most security-relevant tools do this. The verbose-log suggestion above is the minimum; a stderr banner is the user-facing version.

## Risk

Medium-low. The default is `false` (verified by `config.test.ts:830-836`), so users who don't opt in are unaffected. The risk is concentrated in users who *do* opt in but don't realize the global dispatcher carve-out also disables verification for MCP / telemetry / future web-fetch traffic — that's a foot-gun the audit-log nit would mitigate. The proxy + insecure interaction was the highest-risk surface and it's tested explicitly.

The `NODE_TLS_REJECT_UNAUTHORIZED=0` parity is a good UX choice but it's also a backward-compat trap: existing users who set this for an unrelated reason (some other Node tool in their shell init) will silently start running Qwen Code in insecure mode. The audit-log warning is the bare minimum to make this discoverable.

## Verdict

**merge-after-nits** — well-shaped feature, comprehensive tests, documentation lands in the right place. Three soft nits before merge:
1. Add a `verbose_proxy_logger.warning` (or stderr banner) at startup when `this.insecure === true`, naming the trigger.
2. One-line comment above the `NODE_TLS_REJECT_UNAUTHORIZED === "0"` check explaining the strict-equality contract.
3. Open a follow-up issue for `QWEN_TLS_INSECURE_HOSTS` allowlist (per-host carve-out for v2).

The optional Claude-Code-parity behavior (`NODE_TLS_REJECT_UNAUTHORIZED=0`) is the user-facing win that makes this PR worth landing now rather than waiting for the per-host version.

## What I learned

The "fetch ignores `NODE_TLS_REJECT_UNAUTHORIZED`" footgun is one of the most under-documented gotchas of Node 18+. It catches every team that migrates from `node-fetch` or `axios` to native fetch, because the env var has worked since Node 0.10. The Anthropic SDK works around it through a custom dispatcher; OpenAI's SDK doesn't bother. The right fix isn't "tell users to use a different SDK" — it's exactly what this PR does: thread the option through every dispatcher you construct, including the global one. The pattern (`connect: { rejectUnauthorized: false }` on `Agent` / `ProxyAgent`) is the same three lines everywhere; the discipline is in *finding all the dispatcher construction sites*. This PR found four (default OpenAI, DashScope OpenAI, Anthropic, global) which matches my read of the code. If a fifth shows up later (custom MCP transport, image/audio provider), it'll need the same `getInsecure()` thread-through.
