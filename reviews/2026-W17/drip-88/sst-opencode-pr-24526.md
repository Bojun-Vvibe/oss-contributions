---
pr: 24526
repo: sst/opencode
sha: 391fff01949fc514dd028a7bef26177bb7ab5fa0
verdict: needs-discussion
date: 2026-04-27
---

# sst/opencode #24526 — Simplify Infinity mode judge prompt and reduce logging noise

- **Author**: greatbody
- **Head SHA**: 391fff01949fc514dd028a7bef26177bb7ab5fa0
- **Size**: +798/-1 across 11 files spanning `packages/app`, `packages/opencode/src/agent`, `packages/opencode/src/cli/cmd/tui`, `packages/opencode/src/server/routes/instance/session.ts`, `packages/opencode/src/session/prompt.ts`, and the V2 SDK gen output.

## Scope

The PR title and bullets describe four small things: simplify the judge system prompt, remove debug logging around judge invocation, drop the `reason` field from judge responses, and stop injecting `[Judge decided to stop]` metadata messages. The actual diff is **+798/-1** because it bundles the *initial implementation of Infinity mode* — a new `infinity` agent (defined in `agent/agent.ts`), a new prompt file (`agent/prompt/infinity.txt`), a new dialog (`tui/component/dialog-session-infinity.tsx`), TUI-route + prompt-input toggle wiring, V2 server routes (`infinitySet`, `infinityClear`, `infinityStatus`), and the regenerated SDK types in `packages/sdk/js/src/v2/gen/{sdk,types}.gen.ts`. The `-1` is the only deletion; this is essentially a new feature with a misleading "simplify" framing.

## Specific findings

- `packages/app/src/components/prompt-input.tsx:1066-1102` — adds `useQuery({queryKey:["infinity",params.id], queryFn: …, refetchInterval: 10000})` plus a `useMutation` toggling between `infinityClear` (when active) and `infinitySet` (when inactive). The query polls every 10s — for a long-running TUI this is 8,640 requests/day per open session against the local server. Cheap, but it's still noise on the bus + server logs that will trigger at #24525's new wildcard-cap accounting. Consider piggy-backing on the existing event stream instead of polling.
- `packages/app/src/components/prompt-input.tsx:1068-1078` — `queryFn` falls back to `{active:false}` on *any* error (`.catch(() => ({ active: false }))`). That's wrong for the user: if `infinityStatus` 500s because the server is misconfigured, the UI silently shows "off" while the user thinks Infinity is enabled. Either surface the error or distinguish "unknown" from "off".
- `packages/app/src/components/prompt-input.tsx:1656-1664` — the toggle button uses `infinityOn() ? "∞" : "∞"` for both states (identical glyph, only the `text-icon-strong-base` vs `text-icon-weak-base` class differs). Looks like a placeholder; pick distinct visual states or document the intent.
- `packages/opencode/src/agent/agent.ts:233-247` — the new `infinity` agent uses `Permission.fromConfig({"*": "deny"})` merged after defaults but before user permissions. The user-merge being last means a user config with `"*": "ask"` would *override* the agent's deny-by-default — is that intended? An autonomous "judge" agent that can be loosened by user permission config is a meaningful security surface; document the precedence or pin the deny.
- `packages/opencode/src/agent/agent.ts:235` — `hidden: true, native: true`. The `hidden` keeps it out of the agent picker; good. But `native: true` typically means "ships with the binary" — verify that hidden+native agents still go through the same prompt-loading + permission-resolution code paths as user-defined agents (test coverage?).
- `packages/opencode/src/session/prompt.ts` — the actual "remove `reason` field" and "stop injecting `[Judge decided to stop]` metadata messages" changes are the smallest part of this PR; no diff in the head-200 confirms how the judge integration deletes those branches. Ask author for line refs.
- `packages/sdk/js/src/v2/gen/{sdk,types}.gen.ts` — regenerated SDK types. These are generator output; verify the generator is checked in and CI re-generates against the spec, otherwise these can drift on the next merge.
- **PR title is wrong**. "Simplify Infinity mode judge prompt and reduce logging noise" suggests a small cleanup; the diff is the *initial Infinity-mode UX implementation* (toggle button, dialog, polling, V2 routes). Reviewers will skim and miss the security-sensitive new agent definition.

## Risk

High, primarily because of the framing. A reviewer who reads only the title/body will believe this is a few prompt-text edits and not notice (a) a new V2 server endpoint surface, (b) a new permission-deny-by-default agent definition, (c) a 10s-polling client, (d) regenerated SDK output. Each of those needs its own review attention.

## Verdict

**needs-discussion** — split this PR. (1) The "simplify judge prompt" + "remove debug logging" + "drop `reason` field" + "stop injecting metadata messages" set is what the title promises; that's a small, safe diff and should land on its own. (2) The Infinity-mode toggle UI + V2 routes + SDK regen + new agent definition is a *feature* PR that needs its own title, design discussion (polling-vs-events, deny-by-default agent precedence), and probably a feature-flag. As-is, the bundle makes both halves harder to reason about and bisect later.
