---
pr-url: https://github.com/sst/opencode/pull/25019
sha: 9bddf7f3ef53
verdict: merge-as-is
---

# fix: handle invalid mcp urls

Defensive `URL.canParse`-then-construct guard at `packages/opencode/src/mcp/index.ts:117-121`, factored into a tiny `remoteURL(key, value)` helper that logs `"invalid remote mcp url"` with the offending key and returns `undefined` on parse failure. Two call sites consume it: the main remote-MCP layer at `:275-281` returns a structured `{client: undefined, status: {status: "failed", error: 'Invalid MCP URL for "$key"'}}` (so the rest of the MCP enumeration continues and the bad server shows up as `failed` in the UI rather than wedging the whole layer), and the OAuth-login path at `:737-738` throws `Invalid MCP URL for "$mcpName"` (correct: OAuth login is an interactive user action, throwing surfaces the error directly to the caller).

Bug shape is the classic "we trusted user-supplied config to construct a `new URL()` and got `TypeError: Invalid URL`": the prior code at `:294/:301/:768` did `new URL(mcp.url)` directly, so a single typo in `opencode.json` (say `"url": "ttps://..."`) would throw at layer-build time, take out the entire MCP layer initialization, and leave the rest of the app with no MCP servers configured — including the *valid* ones. The new shape converts that fail-stop into a per-server fail-soft with a named error, which is exactly the right policy for "user-supplied list of N independent things" (one bad entry shouldn't kill the other N-1).

Two small things, neither gating: (a) the `URL.canParse` guard treats e.g. `mailto:foo@bar` as valid (it parses) but the downstream `StreamableHTTPClientTransport` will fail on it later with a less-helpful error — a `url.protocol === "https:" || url.protocol === "http:"` follow-up check inside `remoteURL` would catch the protocol-class typo at the same surface; (b) the warning log at `:119` doesn't include the actual `value` (just the `key`), which is fine for privacy (URLs can contain bearer tokens in query strings) but means debugging "which url did I have configured wrong?" requires opening the config file — a `value: value.slice(0, 12) + "..."` truncation would split the difference.

## what I learned
For any data structure that's "list of independent items, all derived from user-editable config," the parse step needs to be per-item with a per-item error path, not all-or-nothing at the layer level. The shape `{client: undefined, status: {status: "failed", error: "..."}}` is exactly right because it (1) lets the consumer keep the slot in its own data structures (`Map<key, item>`), (2) carries the failure reason for UI display, and (3) doesn't require the caller to know ahead of time which keys might fail. The alternative — silently dropping the bad entry — would produce the worst possible UX: "I configured 5 MCP servers and only 4 show up, with no error, anywhere."
