# continuedev/continue #12212 — fix(bedrock): opt into httpBearerAuth when apiKey is set

- **Repo**: continuedev/continue
- **PR**: [#12212](https://github.com/continuedev/continue/pull/12212)
- **Head SHA**: `aef2b14e6bdc2d57e66bca3853d08aab54ebab15`
- **Author**: TianyiChen
- **State**: OPEN (+2 / -1)
- **Verdict**: `request-changes`

## Context

Bedrock added a long-lived API key authentication mode (you mint
a "Bedrock API Key" in the AWS console and use it as a Bearer
token instead of SigV4 signing). The Continue Bedrock provider
config documented at [continue.dev/customize/.../bedrock] supports
`apiKey: ${{ secrets.BEDROCK_API_KEY }}`, but the underlying AWS
SDK call falls through to its credential chain because the
SDK doesn't auto-pick the bearer auth scheme just because a
`token` provider is set. Result: `Could not load credentials from
any providers`.

## Design

Two-line change at `core/llm/llms/Bedrock.ts:78-81`:

```ts
token: async () => ({ token: this.apiKey! }),
+ authSchemePreference: ["httpBearerAuth"],
} as any);
```

Adds `authSchemePreference: ["httpBearerAuth"]` to the
`BedrockRuntimeClient` config inside the `if (this.apiKey)`
branch. The cast `as any` is needed because the SDK type
definitions for `BedrockRuntimeClientConfig` don't expose
`authSchemePreference` in the version Continue is pinned to.

## Risks

- **`as any` is a code smell here**, not a fix. The right path
  is one of: (a) bump the AWS SDK to a version where
  `authSchemePreference` is in the public type, (b) extend the
  config type with a local interface that adds the field, or
  (c) use a more targeted cast like `as BedrockRuntimeClientConfig
  & { authSchemePreference?: string[] }`. Option (c) is the
  smallest scope and survives an SDK bump that adds the field
  natively.
- **No test added.** The bug is a missing field in a single
  branch, and the failure mode is "the SDK falls back to SigV4
  and 401s." A unit test that constructs the client with
  `apiKey: "x"` and asserts the resolved config has
  `authSchemePreference: ["httpBearerAuth"]` would prevent
  regression.
- **Forgetting `httpBearerAuth` for sibling providers.** The
  `Bedrock.ts` constructor has at least two code paths
  (apiKey vs IAM at line 82+ — visible in the diff context).
  The fix is correctly scoped to the apiKey branch only, but
  if Continue has a separate `BedrockRuntime` import elsewhere
  (e.g. for Anthropic-on-Bedrock or for embeddings), each
  needs the same treatment. Worth a grep pass.
- **Behavior change for users who set `apiKey` plus IAM creds**:
  before this PR, the SDK would error out and the user might
  notice their IAM creds were being ignored. After this PR,
  the apiKey path silently wins. That's almost certainly the
  intended behavior, but worth documenting in the bedrock
  provider docs.
- **`BEDROCK_API_KEY` lifecycle**: AWS Bedrock API keys are
  long-lived bearer tokens tied to an IAM principal. If the
  user's threat model assumed SigV4-style short-lived creds,
  this fix shifts them to long-lived ones without warning.
  Documentation responsibility, not a code issue, but worth
  flagging in the PR body.

## Suggestions

1. Replace `as any` with a typed extension:
   ```ts
   } as BedrockRuntimeClientConfig & {
     authSchemePreference?: string[];
   });
   ```
2. Add a unit test asserting the field is present in the
   resolved config when `apiKey` is set.
3. Grep for other `BedrockRuntimeClient` / `BedrockClient`
   constructions in the repo and apply the same fix where
   apiKey-mode is supported.
4. Update the bedrock provider docs to mention that setting
   `apiKey` opts into bearer auth and bypasses any IAM creds.

## What I learned

The AWS SDK v3's auth-scheme resolution is signature-based: it
looks at *which* credential providers are configured to decide
which auth scheme to use. Setting `token: async () => ({...})`
alone isn't sufficient because the SDK's default scheme list
still puts SigV4 first. `authSchemePreference: ["httpBearerAuth"]`
is the explicit override. This is a recurring footgun for any
code that wraps AWS SDK clients and exposes a config knob —
worth a `// AWS SDK requires explicit scheme preference for
bearer auth; see aws-sdk-js-v3#XXXX` comment so the next
maintainer doesn't strip it as redundant.
