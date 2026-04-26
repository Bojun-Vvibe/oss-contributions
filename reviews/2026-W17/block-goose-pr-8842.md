# block/goose #8842 — feat: lifecycle hooks system

- **Repo**: block/goose
- **PR**: #8842
- **Author**: michaelneale (Michael Neale)
- **Head SHA**: 83c0dcfebf305c8843e4263c716aa0475a5ef401
- **Base**: main
- **Size**: +550 / −10 — bulk in new `crates/goose/src/hooks.rs`
  (+~444), with 6 emit sites added across `crates/goose/src/agents/agent.rs`
  (+~80) and a sprinkle of glue in the agent constructor and reply loop.

## What it changes

Adds a config-driven lifecycle hooks subsystem with 6 events:
`before_tool_call`, `after_tool_call`, `on_session_start`,
`on_session_end`, `before_reply`, `after_reply`
(`crates/goose/src/hooks.rs:14-25`). Each `HookEntry` pairs an
optional `matcher` regex (matched against tool name) with one or
more `HookHandler`s, each of which is either a shell `command`
that receives the JSON `HookContext` on stdin, or an HTTP `url`
that receives it as a POST body, with a `timeout` (default 10s).

Hooks are loaded from `goose config.yaml` via
`HookManager::from_config()` (`hooks.rs:107-116`). The agent
constructor (`agent.rs:251-255`) instantiates a manager, and three
emit sites are wired:

- `before_tool_call` (`agent.rs:594-624`): synchronously gates the
  tool dispatch — a hook returning `block: true` short-circuits
  with `INVALID_REQUEST` and the supplied reason.
- `after_tool_call` (`agent.rs:692-708`): fire-and-forget, no
  blocking semantics.
- `before_reply` (`agent.rs:1118-1135`): emitted at the top of the
  reply loop with the user message text in `HookContext.message`.

## Strengths

- Event enum is exhaustive and `serde(rename_all = "snake_case")`
  (`hooks.rs:16`) so the YAML keys are predictable and unsurprising.
- The `HookDecision` short-circuit at `hooks.rs:218-221` returns
  on the *first* `block: true` hook — important so that an
  early-deny rule can't be silently overridden by a later
  permissive rule. Matches the principle-of-least-surprise for an
  authorization layer.
- `has_hooks()` gate at `agent.rs:594, 694, 1119` avoids the
  serialization cost of building `HookContext` when no hooks are
  registered for that event. With the hash-map check at
  `hooks.rs:163-167` returning `false` on empty/missing entries,
  the no-hooks code path is essentially free.
- Matcher regex compilation failure is logged and the hook is
  *skipped*, not panicked (`hooks.rs:194-200`). Right call —
  one bad regex shouldn't take down all the other hooks for that
  event.
- Timeout default of 10s (`hooks.rs:217`) is sane for a synchronous
  `before_tool_call` gate; long enough for a network-bound policy
  check, short enough that a stuck script can't hang the agent.

## Concerns / asks

- **`after_tool_call` is documented as "fire-and-forget" but
  `agent.rs:707-708` actually `await`s it.** That means a slow
  after-hook still blocks the reply loop just like a before-hook.
  Either:
  1. Spawn it via `tokio::spawn` so the loop returns immediately
     (matching the docstring), or
  2. Update the docstring and the comment at `:693` to say
     "synchronously emitted; respects timeout".
  As-is, the contract and the implementation disagree, and the
  consequence is that a misconfigured after-hook with a 30s
  timeout will pause the chat between every tool call.
- `tool_result: None` at every emit site (`agent.rs:601, 700,
  1126`) — the `HookContext` *has* a `tool_result` field but
  no caller populates it. For `after_tool_call` this is the most
  valuable field a hook would want (e.g. "log the result of every
  shell command", "block on PII in tool output"). The result is
  available right there at `agent.rs:692` (just after the
  `WAITING_TOOL_END` debug log). Wire it through.
- The `before_tool_call` block at `agent.rs:614-622` returns
  `INVALID_REQUEST` with the reason in the error message. That
  means the *model* sees the block reason in its tool-call result,
  which is the right behavior for a guardrail — but the reason is
  also the only thing logged. For audit purposes (a hook denied a
  tool call), worth emitting a structured tracing event with the
  `tool_name`, `session_id`, and `reason` fields rather than just
  the existing `tracing::info!` line at `:611`.
- `Config::global()` at `hooks.rs:121` is a process-wide singleton.
  For a long-running `goose serve` whose config file gets edited
  at runtime, the hooks won't reload — `HookManager::from_config()`
  is only called once in the agent constructor. Worth either
  documenting "edit config, restart goose serve" or wiring a
  config-watcher reload path.
- Two handler types (`command` shell + `url` HTTP) but
  `HookHandler` (`hooks.rs:53-60`) makes both `Option<String>` —
  nothing prevents a config from setting *both* `command` and
  `url`, or *neither*. Convert to a tagged enum or at least
  validate at load time and surface a config error, otherwise
  ambiguous configs fail silently or behave inconsistently.
- The `matcher` regex at `hooks.rs:189-199` is recompiled on
  every emit. For a high-frequency tool-call workload, that's
  measurable. Compile once at config-load time and store the
  `Regex` in `HookEntry`.
- No tests in this PR. For a feature that:
  - executes user-supplied shell commands,
  - can block tool dispatch,
  - has a non-obvious "first block wins" precedence rule,
  - has a regex-matching subsystem with skip-on-error semantics,
  the absence of any unit tests is a real gap. At minimum:
  - block-on-first-deny precedence,
  - regex-skip-on-invalid behavior,
  - timeout enforcement,
  - per-event has_hooks gating.

## Verdict

**request-changes** — the design and the integration points are
sound, but four concrete issues need addressing before merge:
(1) the `after_tool_call` docstring/impl disagreement,
(2) `tool_result` never populated,
(3) `command`/`url` mutual-exclusivity not enforced,
(4) zero tests for a security-sensitive subsystem.

The matcher recompilation, structured audit logging, and
config-watcher reload can be follow-ups, but tests for the
deny-precedence path and the regex-skip behavior should land
with this PR — those are the cases an operator would assume
Just Work, and they're cheap to cover.

## What I learned

A "lifecycle hooks" feature looks like a UX feature but is
actually a security/policy feature: any hook with `block: true`
is implicitly an authorization layer. That changes the bar for
review — config malformation, error handling on hook failure,
audit logging, and ordering semantics all become
correctness-critical, not nice-to-have. The same pattern recurs
in MCP servers, agent permissions, and webhook-style policy
plugins; the temptation to "just call user code at lifecycle
points" almost always evolves into a guardrail/audit subsystem,
and it's cheaper to design for that on day one.
