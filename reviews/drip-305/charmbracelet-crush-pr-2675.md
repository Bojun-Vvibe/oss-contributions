# charmbracelet/crush PR #2675 — Fix command aliases/args parsing and empty tool-call input normalization

- **Author:** axeprpr
- **Head SHA:** `a0bd0773aa018de1128a08dbeb57bbcf90368940`
- **Base:** `main`
- **Size:** +62 / -3 (5 files)

## What it does

Three independent fixes bundled into one PR:

1. Add an `exit` command-dialog alias mapped to the same `tea.QuitMsg` as
   `quit`.
2. Allow Unicode placeholders (`$数据目录`, `$ДАННЫЕ_1`, `$ÅR2`) in user
   commands by widening the regex from ASCII upper-snake.
3. Normalize empty / whitespace-only tool-call inputs to `{}` before
   replaying or sending message history, to prevent malformed-payload retry
   loops.

## Diff observations

- `internal/agent/agent.go:387` — wraps `tc.Input` with
  `normalizeToolCallInput(tc.Input)` when reconstructing `message.ToolCall`
  during replay.
- `internal/agent/agent.go:618-624` — adds the local
  `normalizeToolCallInput` helper:

  ```go
  func normalizeToolCallInput(input string) string {
      if strings.TrimSpace(input) == "" {
          return "{}"
      }
      return input
  }
  ```

- `internal/message/content.go:145-151` — duplicates the same helper in the
  `message` package; `internal/message/content.go:540` calls it for
  `fantasy.ToolCallPart.Input` in `ToAIMessage`. So the same call
  (`tc.Input` / `call.Input`) is now normalized at *both* the replay
  boundary in `agent` and the outbound boundary in `message`.
- `internal/commands/commands.go:17` — regex change:
  ```diff
  -var namedArgPattern = regexp.MustCompile(`\$([A-Z][A-Z0-9_]*)`)
  +var namedArgPattern = regexp.MustCompile(`\$([\p{L}_][\p{L}\p{N}_]*)`)
  ```
- `internal/ui/dialog/commands.go:527` — adds `NewCommandItem(... "exit",
  "Exit", "", tea.QuitMsg{})` directly above the existing `quit` entry.
- `internal/agent/agent_test.go:797-803` — `TestNormalizeToolCallInput`
  covers `""`, whitespace, and a real JSON payload.
- `internal/message/content_test.go:144-165` —
  `TestToAIMessage_EmptyToolCallInputIsNormalized` asserts the
  `fantasy.ToolCallPart.Input` is `"{}"`.
- `internal/commands/commands_test.go:55-65` —
  `TestExtractArgNames_UnicodePlaceholders` checks 4 distinct args (Latin,
  CJK, Cyrillic, Latin-with-diacritic) and verifies dedup by repeating
  `$DATA_DIR`.

## Strengths

- Each fix has a focused unit test, all in the right package.
- The Unicode regex change uses `\p{L}` / `\p{N}` rather than a hand-rolled
  block list — correct way to do this in Go's `regexp/syntax` (RE2).
- The empty-input normalization happens at both the replay site and the
  outbound `ToAIMessage` site; if a stale empty payload is already in
  history, it still gets fixed on the way to the provider.

## Concerns

- The helper is duplicated verbatim in `internal/agent/agent.go` and
  `internal/message/content.go`. Both versions have identical bodies. Lift it
  into one of the two packages (probably `message`, since the data
  originates there) and import it from the other to avoid drift.
- Three unrelated fixes in one PR makes bisecting and reverting harder. Not
  a blocker for a small repo, but the `exit` alias really is independent of
  the regex change which is independent of the tool-call normalization — a
  reviewer can't approve any one of them without implicitly approving all
  three.
- The Unicode regex broadening is a behavior change for any existing
  user-command file that contained a `$` followed by a non-ASCII letter
  *without intending it to be a placeholder*. The risk is low but real;
  worth a one-line note in CHANGELOG / docs.
- `tea.QuitMsg{}` as an action mirrors what `quit` does, but if the dialog
  ever switches to a different quit primitive (e.g. confirm-quit), both
  entries will need to be updated together. A shared constant would prevent
  drift.

## Nits (non-blocking)

- `TestExtractArgNames_UnicodePlaceholders` doesn't test `$0DIGITSTART`
  rejection — the new regex still requires `[\p{L}_]` first, but a quick
  negative case would lock that in.

## Verdict

**merge-after-nits** — recommend splitting into 2–3 PRs (or at minimum
de-duplicating `normalizeToolCallInput` into one package) before merge;
otherwise the changes themselves are well-tested and correct.
