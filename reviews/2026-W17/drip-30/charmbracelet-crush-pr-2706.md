# charmbracelet/crush#2706 — Update tooling notes to give agent some better git and github instructions

**Verdict: merge-after-nits**

- PR: https://github.com/charmbracelet/crush/pull/2706
- Author: taigrr (Tai Groot)
- Base SHA: `fa13c7fcadf670eac16e59c73d0bd82cedc43691`
- Head SHA: `1e5cf43bdf54e8874e8fc06550580cf3a1b5c6da`
- +66 / -11

## Summary

Two changes bundled. (1) Adds a guidance line to the bash tool's
embedded prompt template advising the agent to use `-m` or
`GIT_EDITOR=true` when rebasing (avoids interactive editor hangs).
(2) Threads a `gh`-availability check through the fetch / web_fetch /
web_search tool descriptions: detects `gh` on PATH at startup and
appends a "use `gh` CLI in bash instead" hint to the relevant tool
descriptions when present. To support templating, renames
`bash.tpl → bash.md.tpl`, `fetch.md → fetch.md.tpl`,
`web_fetch.md → web_fetch.md.tpl` and adds a shared
`renderToolDescription` helper in `tools.go`.

## Specific findings

- `internal/agent/tools/tools.go:+22` (head SHA
  `1e5cf43bdf54e8874e8fc06550580cf3a1b5c6da`):
  ```go
  var ghAvailable = func() bool {
      _, err := exec.LookPath("gh")
      return err == nil
  }()
  ```
  `gh` availability is captured **once at package init**, not per-call.
  This is fine for the typical case but means the agent will continue
  to advertise the `gh` hint for the lifetime of the process even if
  `gh` was uninstalled mid-session, and conversely won't pick up a
  fresh install. Acceptable for tool descriptions (which are baked
  into the system prompt), but worth a comment noting the lifecycle.
- `internal/agent/tools/bash.md.tpl:+1/-1`:
  ```
  Notes: ... no -i flags, no empty commits, return empty response,
  when rebasing always use -m or GIT_EDITOR=true.
  ```
  The `GIT_EDITOR=true` advice is correct (`true` exits 0 immediately,
  bypassing the editor for any git op that wants one). Only nit: the
  `-m` flag doesn't apply to `git rebase` itself — `-m` is a `git
  commit` / `git merge` flag. For rebase the right escape is
  `GIT_SEQUENCE_EDITOR=true` (for interactive rebase) or
  `GIT_EDITOR=true` (for non-interactive editor invocations during
  conflict resolution). The current wording will mislead the agent
  into trying `git rebase -m` which will trigger merge-strategy
  semantics, not skip the editor. Worth tightening to
  `"when rebasing, set GIT_EDITOR=true (and GIT_SEQUENCE_EDITOR=true
  for interactive rebases) to bypass the editor"`.
- `internal/agent/tools/fetch.md.tpl:+4` and `web_fetch.md.tpl:+3`
  use a `{{- if .GhAvailable }}` template directive to conditionally
  emit the `gh` hint. The `-` whitespace trim is correct so the
  resulting markdown stays well-formed when the directive is omitted.
- `internal/agent/tools/fetch.go:+8/-3` and `web_fetch.go:+8/-3` —
  both files switch from `_ "embed"` + raw `[]byte` to
  `html/template` parsing at package init via
  `template.Must(template.New(...).Parse(...))`. The use of
  `html/template` (not `text/template`) is mildly suspect — these
  outputs are markdown going into LLM prompts, not HTML. `html/template`
  will HTML-escape any `<`/`>`/`&` in the data, which here is just
  the `GhAvailable` bool, so practically harmless, but `text/template`
  would be the correct package choice on principle.
- `tools.go:+22` shared helper `renderToolDescription` — single
  rendering helper used by all three tools, good DRY. The
  `panic("failed to execute tool description template: " + err.Error())`
  on render failure is reasonable for a templating bug that should
  never happen at runtime, but worth noting it converts a startup-time
  template-data mistake into a hard process abort instead of a
  graceful tool-disable.

## Rationale

The intent is right and the architecture (template-render at startup,
`gh` detection cached at init, conditional hint blocks) is clean. Two
substantive nits keep this from straight merge-as-is. First and most
important, the rebase advice text in `bash.md.tpl` has the wrong
flag: `git rebase -m` doesn't bypass the editor — `-m` on rebase is
the merge-strategy switch (`--merge`), and using it will produce
semantically different rebase behavior. The agent will then either
work around it correctly (and ignore our advice) or follow our advice
and produce surprising rebases. Right answer is `GIT_EDITOR=true` for
the non-interactive case and `GIT_SEQUENCE_EDITOR=true` for
interactive rebase. Second, lower priority: `html/template` should
probably be `text/template` for markdown output — the HTML-escaping
won't break anything with just a bool variable, but if anyone later
adds a string field to `toolDescriptionData` containing an `&` or
`<`, the rendered description will get gratuitously escaped. Not
worth blocking on, but worth flipping while it's a one-import-line
change. Third, the 1-shot `ghAvailable = func() bool { ... }()` at
package init means installing `gh` mid-session won't surface the
hint; that's acceptable because the tool description is baked into
the system prompt (which is also captured per-session), so the
behaviors are at least consistent. The bash template change for git
commits doesn't drift far from the existing 80+ line embedded prompt.
Land after fixing the rebase flag wording.
