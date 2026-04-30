# sst/opencode PR #25057 — tweak: make azure onboarding ux a bit better

- PR: https://github.com/sst/opencode/pull/25057
- Head SHA: `215afcfad16df8c2e23f8ae1b800ecbfd52883a4`
- Files touched: `packages/opencode/src/plugin/azure.ts` (+26/-0, new), `packages/opencode/src/plugin/index.ts` (+2/-0)

## Specific citations

- New `AzureAuthPlugin` at `azure.ts:3-26` registers itself under `provider: "azure"` with a single `methods[0] = { type: "api", label: "API key", prompts }` entry; the `prompts` array is conditionally populated at `:4-12` only when `process.env.AZURE_RESOURCE_NAME` is unset, asking the user for `resourceName` with placeholder `e.g. my-models` and message `"Enter Azure Resource Name"`.
- Wired into `INTERNAL_PLUGINS` at `plugin/index.ts:62-65` (import at `:20`, append at the end of the existing internal plugin list right after the two Cloudflare plugins).

## Verdict: merge-after-nits

## Concerns / nits

1. **Resource name only — what about API key?** The plugin shape declares `type: "api"` but the `prompts` array contains *only* the resource-name prompt. If the user has `AZURE_RESOURCE_NAME` set in env but no API key, the wizard now collects nothing and presumably falls through to the framework's default API-key prompt. If the framework *doesn't* auto-prompt for the api-key on `type: "api"`, then with the env var pre-set the user is silently stuck. Worth a one-line confirmation in the PR body or an explicit `apiKey` prompt entry beside the conditional resource-name one.
2. **Empty-prompts edge case at `:13-23`.** When `AZURE_RESOURCE_NAME` is set, `prompts: []` is returned in the methods array. Confirm the renderer treats an empty `prompts` array as "no extra fields" rather than throwing — a quick smoke test under `AZURE_RESOURCE_NAME=foo` is the cheapest insurance.
3. **No validation on the input.** A typo'd resource name (e.g. trailing slash, `https://` prefix) will silently fail at first request time with an opaque DNS error. Adding `validate: (v) => v.match(/^[a-z0-9-]+$/) || "..."` (Azure naming rules) would catch the most common shapes. Non-blocking.
4. **No PR body / motivation.** The diff is mechanical and obviously correct, but the PR title says "make azure onboarding ux a bit better" — readers can't tell from the diff alone what UX problem this solves (was the user previously asked for the full URL? Just the API key with no resource name? Something else?). One screenshot or before/after sentence would carry the change.
5. **Insertion order inside `INTERNAL_PLUGINS`** at `plugin/index.ts:65` puts Azure last; the order isn't documented as load-bearing but if there's any "first matching provider wins" semantic in the auth dispatcher, this position should be confirmed deliberate vs incidental.
