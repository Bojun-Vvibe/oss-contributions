# sst/opencode#24595 — fix(opencode): don't override User-Agent set via provider options.headers

- PR: https://github.com/sst/opencode/pull/24595
- Head SHA: `a824489`
- Diff: +0 / -1 in `packages/opencode/src/session/llm.ts`
- Closes #24594

## What it does
Removes the line `"User-Agent": \`opencode/${InstallationVersion}\`` from the third-party-provider branch of the headers object in `session/llm.ts:381`. The opencode-hosted branch (the `if` arm) is untouched. The author's analysis is precise: AI SDK's `combineHeaders` merges the call-level header *after* `getHeaders()`, so a user-configured UA in `provider.<name>.options.headers` was being silently clobbered.

## Strengths
- Surgical: literally one line. Risk surface is tiny and isolated to non-opencode-hosted providers.
- Reproduction in the PR description (corp LiteLLM gateway returning 403 because UA allowlist never saw the configured value) is concrete and credible. The captured "before" UA `opencode/1.14.28 ai-sdk/provider-utils/4.0.23 runtime/bun/1.3.13` matches what one would see if the offending line were active.
- `withUserAgentSuffix` chain is preserved on the post-fix path, so opencode/AI-SDK still leave a UA breadcrumb after the user's value — no telemetry loss for anyone who was relying on it.

## Concerns
- **No regression test.** A 1-line behavior change against an already-buggy code path deserves a unit test that constructs the third-party header chain and asserts the UA round-trips. Without it, the next refactor of `headers` ordering can silently re-introduce the bug.
- **Default UA no longer set for third-party providers.** Users who *don't* configure a UA now send only the AI-SDK suffix chain. That's almost always fine, but some upstream gateways reject requests without a recognizable UA prefix. A safer pattern: keep `"User-Agent": ...` but apply it via `combineHeaders(defaults, userHeaders)` so user values still win. The current diff trades one bug for a (small) behavior regression for users who weren't customizing UA.
- The `if` arm — opencode-hosted — still hard-sets `User-Agent` (line `:374`). If we agree opencode should respect user UA configs, the same logic applies there too. Worth deciding if the inconsistency is intentional (telemetry on hosted) or an oversight.

## Verdict
**merge-after-nits** — the fix is right; ask for a small unit test pinning the header-merge order, and a one-line note in the PR clarifying the opencode-hosted asymmetry is intentional.
