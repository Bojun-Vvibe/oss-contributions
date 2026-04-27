# sst/opencode #24537 — fix(edit): include args in tool output to prevent crash (#24529)

- Author: PratikRai0101 (Pratik Rai)
- Head SHA: `16612ada5174ae35d0c6ca98b77b617aa89a0b97`
- Single file: `packages/opencode/src/tool/edit.ts` (+1 / −0)

## Specifics

- `edit.ts:206` adds `args: params,` to the tool-result object returned alongside `metadata`, `title`, and `output`. The fix follows the existing return-shape pattern that other tools (read/grep/glob in this same package) use to round-trip the original arguments back to the renderer.
- Linked issue #24529 reports a TUI render crash when the edit tool result lacks an `args` field — the renderer presumably destructures `result.args.filePath` (or similar) and throws on `undefined`. The one-liner restores the field the renderer expects.

## Concerns

- The fix is the right shape but **does not test the regression**. A two-line snapshot or unit test asserting `result.args.filePath === <input>` and `result.args.oldString === <input>` would lock this down so the next refactor of the return-shape doesn't silently regress #24529 again.
- `params` is the entire input parameter object including potentially-large `oldString` / `newString` payloads. Echoing them back to the client doubles the wire size of every edit tool result. Worth a one-line comment noting that the renderer requires this and that a slimmer `{ filePath: params.filePath }` projection would be preferable if/when the renderer contract allows.
- No upstream type definition shown in the diff — verify the `ToolOutput` / equivalent type already permits an `args` field, otherwise this will fail typecheck. The PR description claims `bun typecheck` was run; trust but verify.

## Verdict

`merge-after-nits` — correct minimal fix for a real crash, but ship with a regression test and consider the payload-size note. If the maintainer prioritises shipping the hot-fix, `merge-as-is` is defensible.

