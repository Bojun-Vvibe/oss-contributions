# charmbracelet/crush PR #2706 — Update tooling notes to give agent some better git and github instructions

- Head SHA: `f74d968963f344367312031d95df83cd3c58d014`
- Size: +71 / -11, 9 files

## Specific refs

- `internal/agent/tools/tools.go:+26` — adds `ghAvailable` (computed once via `exec.LookPath("gh")`), `toolDescriptionData`, and `renderToolDescription()` helper for templated descriptions.
- `internal/agent/tools/bash.go:56,69,153` — renames `bash.tpl` → `bash.md.tpl`, threads `RgAvailable: getRg() != ""` into the template data.
- `internal/agent/tools/bash.md.tpl:24-26` — new conditional usage hint to prefer `rg` over `grep` when ripgrep is on PATH.
- `internal/agent/tools/bash.md.tpl:88` — git-rebase guidance updated: "always use -m or `GIT_EDITOR=true`" to prevent agent from getting stuck on interactive rebase editor.
- `internal/agent/tools/fetch.md.tpl:18-20` (formerly `fetch.md`) — adds `{{- if .GhAvailable }}` block redirecting GitHub URL fetches to `gh` CLI when available.
- Same templating pattern applied to `web_fetch.md.tpl` and `web_search.md.tpl`.

## Assessment

Sensible upgrade: descriptions become host-aware without baking environment assumptions into prompts. The `ghAvailable` / `getRg() != ""` checks happen at process-start, not per-call, so cost is negligible.

Concerns:
1. `tools.go:84` uses `html/template` to render plain markdown descriptions. This silently HTML-escapes any `<`, `>`, `&` that appears in the template. The current templates don't use those chars in conditional blocks, but a future edit that adds a code sample with `<-` (Go channel op) or a shell redirect will get mangled. Should be `text/template`.
2. `bash.go:56` uses `html/template` too via the existing import; same hazard for the rg conditional.
3. `ghAvailable` is captured at import time, so a `gh` install later in the same session won't be picked up. Fine for typical agent sessions.

verdict: merge-after-nits