# openai/codex PR #19176 — Add network proxy prompt guidance

- **Repo:** openai/codex
- **PR:** [#19176](https://github.com/openai/codex/pull/19176)
- **Head SHA:** `5e134b6fe5d977a3370db4420ba2cd3b4d13377c`
- **Size:** ~+30/-0 across 3 files
- **Reviewer:** Bojun (drip-23)

## Summary

Adds a small new prompt fragment (`network_proxy.md`) to the
`PermissionsInstructions` system-prompt builder so the model is told
that a managed network proxy may be active for shell commands, and
specifically: "use normal network tools without clearing or overriding
proxy-related environment variables. If a required host is not allowed,
request additional network permissions instead of working around the
proxy."

Pairs with #19184 (the deferred-proxy-denial code change reviewed in
drip-9) — that PR taught the runtime to surface late-arriving Guardian
denials; this PR teaches the model not to *cause* them by stripping
`HTTP_PROXY`/`https_proxy` env vars when running curl/wget/etc.

## What's changed

Three files (full diff inspected):

### 1. `core/src/context/permissions_instructions.rs` (+3 lines)

Diff lines 4–5:
```rust
const NETWORK_PROXY: &str = include_str!("prompts/permissions/network_proxy.md");
```
Plus diff lines 14–16: an unconditional `append_section(&mut text,
NETWORK_PROXY.trim_end())` injected immediately after the sandbox
section in the prompt builder. **Always emitted**, regardless of
whether `network_access` is `Restricted` or `Full`.

### 2. `core/src/context/permissions_instructions_tests.rs` (+21 lines)

Diff lines 22–43: a new `includes_network_proxy_guidance` test that
constructs a `PermissionsInstructions` with `NetworkAccess::Restricted`
and asserts the body contains:
- `"# Network Proxy"`
- `"proxy-related environment variables"`
- `"request additional network permissions"`

These are reasonable smoke checks — they pin the load-bearing phrases
the runtime is counting on the model to honor.

### 3. `core/src/context/prompts/permissions/network_proxy.md` (new, +5 lines)

Diff lines 49–53. Two paragraphs. The first sets context ("a managed
network proxy may be active for model-initiated shell commands. When it
is active, Codex applies proxy environment variables automatically");
the second is the rule list ("Honor any `<network>` allow/deny entries
in the environment context. Use normal network tools without clearing
or overriding proxy-related environment variables. If a required host
is not allowed, request additional network permissions instead of
working around the proxy.").

## Concerns

1. **Unconditional emission ignores `NetworkAccess::Full`**

   The append at diff line 15 is unconditional. But the prompt text
   says "A managed network proxy *may* be active" — fine, but for users
   who have explicitly granted `network_access = Full` and have no
   Guardian / proxy active, this paragraph is dead text that costs ~80
   tokens per turn forever. Two cleaner options:

   - gate the append on `network_access == Restricted` (cheap; matches
     the test setup);
   - keep unconditional but make the language load-aware ("If a
     managed network proxy is active...") so it's a no-op for users
     without a proxy.

   The current shape is the worst of both worlds: paid for on every
   turn, unclear to the model whether it's relevant.

2. **No mention of `NO_PROXY` / `no_proxy`**

   The instruction "do not clear or override proxy-related environment
   variables" is good but incomplete. The most common way models
   accidentally bypass a proxy is by setting `NO_PROXY=*` to "fix" a
   curl failure, not by clearing `HTTP_PROXY`. Worth adding "this
   includes `NO_PROXY` / `no_proxy`" explicitly. Otherwise the
   guidance leaves the most popular workaround unblocked.

3. **No instruction on what "request additional network permissions"
   actually means**

   The model is told to request permissions instead of working around
   the proxy, but the prompt doesn't say *how*. Is there a tool name
   to invoke? A specific phrasing the runtime listens for? Without a
   concrete handoff, models will fall back to the same workarounds
   the paragraph forbids. Compare against the
   `request_permissions_tool_enabled` config flag visible in the test
   setup (diff line 32) — if the request mechanism is the
   `request_permissions` MCP tool, the prompt should name it.

4. **Test asserts substring match, not the prompt section ordering**

   The test (diff lines 36–42) asserts three substrings exist in the
   body but does not assert ordering or that `network_proxy.md` is
   appended *after* the sandbox section, where the production code
   places it (diff line 14–15). If a future refactor moves the
   network-proxy paragraph before the sandbox paragraph, the test
   passes silently but the prompt context shifts. Cheap fix: assert
   `body.find("# Network Proxy") > body.find("# Sandbox")` (or
   whatever the exact section header is).

5. **No fixture-based snapshot**

   Codex has a heavy `insta` snapshot habit elsewhere in the prompt
   layer (see `permissions_instructions_tests.rs` history). The new
   section deserves an `assert_snapshot!` of the full body for at
   least one (sandbox_mode, network_access) combination, so reviewers
   can see exactly what the model sees post-change. Substring asserts
   miss formatting drift (missing newline between sections, accidental
   double-blank-line, etc.).

## Verdict

`merge-after-nits` —

- gate the append on `NetworkAccess::Restricted` (or rephrase the
  paragraph to be self-noop'ing when no proxy is active);
- name the actual permission-request tool / mechanism in the prompt
  body, otherwise the "request permissions instead" guidance has no
  follow-through;
- mention `NO_PROXY` / `no_proxy` explicitly in the env-var clause;
- add an `insta` snapshot test of the full body for the
  `Restricted` case so any future drift in the section layout is
  caught.

Concept is sound and pairs cleanly with the runtime work in #19184.

## What I learned

This is the second half of a "agent + runtime" co-design pattern that
keeps appearing in codex: the runtime change (#19184 — deferred proxy
denial cancels the running command) only works if the model is also
trained to not actively undermine it (this PR — don't strip the env
var). Both halves need to land together, but they're often split into
separate PRs reviewed by different people. The reviewer-side risk is
that one half lands without the other, leaving the loop open.
The reviewer fix: when reviewing a runtime-side enforcement PR, search
for a matching prompt-side PR (and vice versa) and link them in the
review comments so they don't drift apart.
