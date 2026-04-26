---
pr: 24499
repo: sst/opencode
title: "fix(cli): save cloudflare account ID and gateway ID during auth login"
url: https://github.com/sst/opencode/pull/24499
head_sha: bf356ea4354ff4ab2ef7f2a1d4d18a90aeae0b25
author: JoaquinGimenez1
verdict: merge-after-nits
date: 2026-04-27
---

# sst/opencode #24499 — actually persist Cloudflare account/gateway IDs during `auth login`

Closes #24498. Previously running `opencode auth login` on `cloudflare` /
`cloudflare-ai-gateway` only printed "remember to set
`CLOUDFLARE_GATEWAY_ID`/`CLOUDFLARE_ACCOUNT_ID`/`CLOUDFLARE_API_TOKEN`"
and stored nothing. Provider-load codepath then required env vars or
`metadata` and failed. This PR prompts for the IDs and writes them into
the API auth record's `metadata` so the existing
`cloudflare-ai-gateway loads with auth metadata` codepath works
end-to-end.

## What the diff actually does

`packages/opencode/src/cli/cmd/providers.ts:461-498` — adds an
account-ID prompt, a gateway-ID prompt (gated to
`provider === "cloudflare-ai-gateway"`), then password prompt for the
key, then:

```ts
await put(provider, {
  type: "api",
  key,
  metadata: {
    accountId,
    ...(typeof gatewayId === "string" ? { gatewayId } : {}),
  },
})
prompts.outro("Done")
return
```

`packages/opencode/test/provider/provider.test.ts:2445-2480` — new test
`"cloudflare-ai-gateway loads with auth metadata"` simulates `auth login`
by setting an api-type auth with `metadata.accountId` and
`metadata.gatewayId`, then asserts `list()` produces a defined entry for
`cloudflare-ai-gateway` *with no env vars set*.

## Observations

1. **`return` placement is right** — without it, control falls through to
   the existing `prompts.password({ message: "Enter your API key" })` at
   `:496-501` and you'd prompt for the key twice. Verified from diff: the
   new branch has its own `prompts.password` and ends with `return`.

2. **Plain `cloudflare` (non-gateway) keeps working.** The gateway prompt
   is gated, but accountId is *unconditional*. That's correct for both
   `cloudflare` (Workers AI requires accountId) and
   `cloudflare-ai-gateway` (gateway needs accountId + gatewayId).

3. **Cancel-handling parity.** Each prompt has its own
   `if (prompts.isCancel(...)) throw new UI.CancelledError()` — matches
   the convention already used at `:495,501`. No UX regression.

4. **Validation is minimal.** `validate: (x) => (x && x.length > 0 ? undefined : "Required")`
   doesn't enforce UUID format for accountId or any shape on gatewayId.
   Cloudflare account IDs are 32-hex chars and gateway IDs are also
   well-defined slugs. Not a blocker — bad input will fail fast at the
   first API call — but a regex check would surface "I pasted my email"
   class typos before persistence. Suggest:
   ```ts
   validate: (x) => (x && /^[a-f0-9]{32}$/i.test(x) ? undefined : "Account ID must be a 32-character hex string")
   ```
   for accountId. (Optional polish.)

5. **`gatewayId: string | symbol | undefined` type is awkward.** The
   `typeof gatewayId === "string"` check at `:495` works, but the
   `symbol` case (a cancel sentinel from `prompts`) was already handled
   by `if (prompts.isCancel(gatewayId)) throw ...` two lines up. So by
   the time we reach the `put()` call, `gatewayId` is either `string` or
   `undefined`. The `typeof === "string"` is defensive but reads as if a
   `symbol` is still possible. Clearer:
   ```ts
   if (gatewayId !== undefined) { metadata.gatewayId = gatewayId as string }
   ```
   Cosmetic.

6. **Test assertion is weak.** `expect(providers[ProviderID.make("cloudflare-ai-gateway")]).toBeDefined()`
   only checks the provider entry exists. It doesn't verify the
   account/gateway ID actually flowed through — that the resulting
   provider config has the right `accountId` / `gatewayId`. Add:
   ```ts
   const cfg = providers[ProviderID.make("cloudflare-ai-gateway")]
   expect(cfg?.options?.accountId).toBe("test-account-id")
   expect(cfg?.options?.gatewayId).toBe("test-gateway-id")
   ```
   (or whatever the loader puts the metadata under). Otherwise this test
   passes even if the loader silently drops `metadata.gatewayId`.

7. **Info message is now misleading.** The `prompts.log.info(...)` at
   :461 still says "Cloudflare AI Gateway can be configured with
   CLOUDFLARE_GATEWAY_ID, CLOUDFLARE_ACCOUNT_ID, and CLOUDFLARE_API_TOKEN
   environment variables." That's true, but post-fix the user is about
   to be prompted for those same values inline, so the info reads as if
   env vars are still required. Reword to "These can also be set via
   environment variables; you can skip the prompts below if you prefer
   that."

## Verdict

**merge-after-nits.** Correct, minimal, has a regression test. The two
nits are: tighten the test to assert metadata propagation (not just
existence), and reword the now-misleading info banner. Both 1-2 line
changes.

## Banned-string scan

None.
