# openai/codex PR #20293 — Support chatgpt library tool

- PR: https://github.com/openai/codex/pull/20293
- Head SHA: `70d1d6d55b799f38637f365511b39dd1c5d756d9`
- Files touched: 1 (`codex-rs/utils/plugins/src/mcp_connector.rs` +5 / -0, net +5 lines).

## Specific citations

- The new constant at `codex-rs/utils/plugins/src/mcp_connector.rs:13-14`: `const ALLOWED_OPENAI_CONNECTOR_IDS: &[&str] = &["connector_openai_library"];` — a single-element allowlist that explicitly carves the `connector_openai_library` ID out of the generic `connector_openai_*` deny-prefix established at `:15` (`const DISALLOWED_CONNECTOR_PREFIX: &str = "connector_openai_";`).
- The carve-out logic at `:28-30` of `is_connector_id_allowed_for_originator`: `if ALLOWED_OPENAI_CONNECTOR_IDS.contains(&connector_id) { return true; }` — early-return *before* both the prefix check and the per-originator deny-list lookup. The pre-existing `DISALLOWED_CONNECTOR_IDS` (line `:5-9`, generic disallow list) and `FIRST_PARTY_CHAT_DISALLOWED_CONNECTOR_IDS` (line `:11-12`, the chatgpt-originator-specific extra disallow with `connector_0f9c9d4592e54d0a9a12b3f44a1e2010`) are both bypassed by the new allow.
- No test file in the diff. No regression entry. No CHANGELOG. The diff is purely the constant + the early-return.

## Verdict: merge-after-nits

## Rationale

This is a 5-line policy change to the connector-ID allow/deny matrix. The headline behavior change is correct in shape: the `connector_openai_*` prefix-deny was originally a coarse-grained "block all OpenAI-internal connectors" rule (presumably to prevent users from wiring up first-party chatgpt connectors that aren't supposed to be reachable from a third-party MCP integration), and this PR opens a single ID — `connector_openai_library` — through that wall to enable the chatgpt library tool. The carve-out is correctly scoped: it's a fixed-set allowlist, not a regex or a prefix, so the blast radius is exactly one connector ID and any future "we need to open another one" requires another explicit code change.

The placement of the early-return at `:28-30` (before both deny-list checks) means `connector_openai_library` is allowed unconditionally — even if it were later added to `DISALLOWED_CONNECTOR_IDS` or to the originator-specific `FIRST_PARTY_CHAT_DISALLOWED_CONNECTOR_IDS`, the allow would still win. This is probably the intent (the allow is the policy decision and the deny lists are the legacy state), but a comment naming the ordering rule would help: something like `// allowlist wins over both the generic and the originator-scoped deny lists by design — see policy doc <link>`. Without that, a future maintainer adding a deny-list entry for `connector_openai_library` would be silently overridden and might not understand why.

The actual nits to clean up before merge:

1. **No test.** The function `is_connector_id_allowed_for_originator` has presumably-existing tests in `mcp_connector.rs` that cover the deny-prefix and the originator-specific extras; the new allowlist branch should have at least one positive test (`assert!(is_connector_id_allowed_for_originator("connector_openai_library", "any-originator"))`) and one negative test (`assert!(!is_connector_id_allowed_for_originator("connector_openai_other", "chatgpt"))`) locking that the carve-out is exactly one ID and not the whole prefix. Without these, a future "let's also allow `connector_openai_research`" patch could mistakenly widen the allow to a prefix and pass CI silently.
2. **No CHANGELOG / wire-doc note.** The PR title says "Support chatgpt library tool" but the diff is pure plumbing — there's no doc anywhere explaining what the chatgpt library tool is, why it needs `connector_openai_library`, or what the trust model is for users connecting to it. A one-paragraph note in `mcp_connector.rs` module docs (or a CHANGELOG entry) would close that gap.
3. The `ALLOWED_OPENAI_CONNECTOR_IDS` constant is plural-named for a one-element array. Fine for now; if the design is "we'll add more over time" the plural is correct, if the design is "this is the only one we'll ever allow" then the single element is misleading and a `const ALLOWED_OPENAI_LIBRARY_CONNECTOR_ID: &str = "connector_openai_library"` would be more honest.

None of these block merge — the change is sound and small — but they're all two-minute fixes that a maintainer-side reviewer should ask for before stamping. The risk-of-not-doing-them is mainly that the carve-out becomes load-bearing and ill-documented if it sits unaddressed for a few months.

## What I learned

The most subtle thing about this kind of allowlist-carve-out-in-a-deny-prefix pattern is the *ordering rule* between the two: when a future maintainer wants to deny something specifically, will the deny win or will the allow win? In this PR the ordering is "allow wins" (the early-return at `:28-30` runs before any deny-list lookup), which is the conservative and easy-to-reason-about choice — but it does mean the system can never deny `connector_openai_library` again without either deleting the carve-out or restructuring the function. A comment naming this invariant explicitly would prevent future confusion. As a general rule, every time a code change adds an early-return into an already-layered allow/deny matrix, the reviewer should ask "what happens when the next maintainer wants to reverse this?" — and if the answer requires editing the early-return rather than adding a new entry to a lower-priority list, the ordering should be documented at the call site.
