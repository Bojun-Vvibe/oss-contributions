# anomalyco/opencode#25987 — fix(opencode): resolve config entry names from config directory

- URL: https://github.com/anomalyco/opencode/pull/25987
- PR: #25987
- Author: JGoP-L (JgoP)
- Head SHA: `e8ef636ef3a04c2118a41f9effd749c82effb7ae`
- State: OPEN  | +27 / -9

## Summary

Closes a real bug where `configEntryNameFromPath` would mis-derive a name when an *ancestor* directory of the config dir happens to be literally named `agent` / `agents` / `command` / `commands` (e.g. `/home/agent/.config/opencode/agents/build.md` returned `.config/opencode/agents/build` instead of `build`). Fix has two layers:

1. Callers (`config/agent.ts:130`, `config/command.ts:47`) now pass `path.relative(dir, item)` instead of the absolute item, scoping the search to the configured root.
2. `entry-name.ts:4-13` rewrites `sliceAfterMatch` to anchor on the *last* `/.opencode/` or `/opencode/` segment in the path before scanning, then picks the smallest `index` (closest to the anchor) and ties broken by longer `searchRoot` length (so `/agents/` wins over `/agent/` for `/agents/build.md`).

## Specific references

- `config/agent.ts:127-130`: drops 4-pattern absolute search (`/.opencode/agent/`, `/.opencode/agents/`, `/agent/`, `/agents/`) → 2-pattern relative search (`agents/`, `agent/`).
- `config/command.ts:46-47`: identical refactor for commands.
- `entry-name.ts:4-13`: new `Math.max(lastIndexOf("/.opencode/"), lastIndexOf("/opencode/"))` anchor, then `.toSorted((a,b) => a.index - b.index || b.searchRoot.length - a.searchRoot.length)[0]`.
- `test/config/entry-name.test.ts:4-12`: one new test covering `/home/agent/.config/opencode/agents/build.md` → `build` against the **old** 4-pattern search root list (i.e. validates the helper alone, not the caller change).

## Concerns / nits

- **Test coverage gap**: the new test asserts the *helper* on the legacy patterns. The actual production call site now passes a *relative* path with the *new* 2-pattern list. Add a second test exercising `configEntryNameFromPath("agents/build.md", ["agents/", "agent/"])` → `build` to lock the load-bearing path. Also add an adversarial test for `agents/agent/build.md` (nested) to confirm the longest-root tiebreaker behaves.
- The anchor-then-search algorithm is a nice idea but `lastIndexOf("/opencode/")` will *also* match `/opencode/` inside e.g. a project subdirectory called `opencode` inside another opencode repo — probably fine since the relative-path callers prevent that, but worth a comment.
- The `agent.ts` and `command.ts` changes do not pass `cwd` to `path.relative`; `dir` is presumably already absolute (the surrounding `load(dir: string)` signature suggests so). Add an assertion or comment to document that contract.

## Verdict

**merge-after-nits (man)** — correct fix for a real bug; needs one more test against the new caller-side pattern set.
