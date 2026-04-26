# sst/opencode PR #24508 — feat: toggle to keep model/agent when switching sessions

- **Repo:** sst/opencode
- **PR:** [#24508](https://github.com/sst/opencode/pull/24508)
- **Head SHA:** see PR
- **Size:** 2 files modified, 1 empty file added (`packages/sdk/js/openapi.json`, 0 bytes — accidental)

## Context

Adds a kv-backed boolean toggle `sync_prompt_context_on_session_switch`
that controls whether `Prompt`'s session-switch effect re-syncs model
and agent from the new session into the prompt. Default `true`
preserves existing behavior. Surfaces the toggle in the Session
command palette as a single command that flips between two titles
("Keep model/agent when switching sessions" ↔ "Restore model/agent
per session").

## What the diff actually does

**`prompt/index.tsx:238-243`** — early return guard:
```ts
syncedSessionID = sessionID

const shouldSync = kv.get("sync_prompt_context_on_session_switch", true)
if (!shouldSync) return

// Only set agent if it's a primary agent (not a subagent)
const isPrimaryAgent = local.agent.list().some((x) => x.name === msg.agent)
```
When the toggle is off, the function returns immediately after
recording `syncedSessionID = sessionID`, which means the
`isPrimaryAgent` branch and the model assignment that follow it are
skipped, but the dedup guard against re-syncing the same session
still updates.

**`session/index.tsx:168`** — adds the kv signal:
```ts
const [syncPromptContext, setSyncPromptContext] =
  kv.signal("sync_prompt_context_on_session_switch", true)
```

**`session/index.tsx:683-693`** — adds a command-palette entry that
flips the boolean and clears the dialog. The command's `title`
inverts based on the current value.

**`session/index.tsx:1067-1068`** — drive-by scrollbar token swap:
```diff
-                  backgroundColor: theme.backgroundElement,
-                  foregroundColor: theme.border,
+                  backgroundColor: theme.background,
+                  foregroundColor: theme.borderActive,
```
This is the same Monokai-scrollbar-visibility fix that was
landed separately as #24507 in drip-83. Either #24507 already
merged and this is a rebase artifact, or it's the same author
double-applying the same fix; in both cases the duplicate hunk
should be dropped from this PR.

**`packages/sdk/js/openapi.json`** — new empty file (0 bytes).
Either an accidental `touch` or a placeholder for a future codegen
step. Should be removed before merge.

## Strengths

- The kv-signal pattern matches the surrounding code (e.g.,
  `diff_wrap_mode`, `animations_enabled`,
  `generic_tool_output_visibility` at `session/index.tsx:163-166`)
  so this slots in cleanly with existing settings UX.
- Default-true preserves prior behavior — no flag flip surprise.
- Single early-return guard at `prompt/index.tsx:241-242` is the
  smallest possible change that achieves the goal.
- Title-flipping in the command palette is a nice touch that makes
  the toggle's current state discoverable without a checkbox UI.

## Risks / nits

1. **Duplicate scrollbar fix at `:1067-1068`** — exact same change
   as #24507 from drip-83 (`backgroundElement`→`background`,
   `border`→`borderActive`). If #24507 has merged, rebase this PR
   to drop the hunk; if not, this PR shouldn't be carrying an
   unrelated theme fix in the first place. Either way the hunk
   isn't justified by the title or body.
2. **Empty `packages/sdk/js/openapi.json` file** — creating a
   zero-byte file at the SDK boundary is a footgun. Tools that
   parse the SDK's openapi spec at startup will now fail with
   "unexpected EOF" instead of "file not found". Drop the file.
3. **Toggle name is awkward** —
   `sync_prompt_context_on_session_switch` reads as a triple
   compound. Consider `keep_model_on_session_switch` or
   `inherit_session_context` for parity with the title text.
4. **`syncedSessionID = sessionID` still updates even when sync is
   off** at `prompt/index.tsx:239`. Double-check that's intentional:
   the side effect of marking the session as "synced" without
   actually syncing means a subsequent toggle-flip during the same
   session won't trigger a catch-up sync. If that's the desired
   semantics ("toggle only takes effect on next session switch"),
   call it out in a comment.
5. **No keybinding suggestion** in the command palette entry —
   `category: "Session"` and `value: "session.toggle.sync_prompt_context"`
   are good but there's no `binding` field. Other Session entries
   like `session.page.up` may have keybindings; align with whichever
   convention is used elsewhere.
6. **No test coverage** for the early-return guard. A unit test
   asserting that with the toggle off, model/agent are *not*
   overwritten on session switch, and with the toggle on they
   are, would lock the contract.
7. **`local.agent.list()` is still called** before the guard? No —
   the guard returns before that line. Good.

## Verdict

`merge-after-nits`

The core change (kv toggle + early-return guard + command palette
entry) is correct, minimal, and matches existing patterns. Three
diff-hygiene items must be cleaned up before merge: drop the
duplicate `:1067-1068` scrollbar hunk (already in #24507), remove
the empty `openapi.json`, and add a one-line comment at
`prompt/index.tsx:239` clarifying the intentional behavior of
updating `syncedSessionID` even when sync is disabled. Once those
are addressed this is a clean small feature.
