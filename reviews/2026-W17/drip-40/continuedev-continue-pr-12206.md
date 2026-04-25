# continuedev/continue #12206 — fix: When AGENTS.md does not exist, the traversal will be interrupted

- **Repo**: continuedev/continue
- **PR**: [#12206](https://github.com/continuedev/continue/pull/12206)
- **Head SHA**: `4f58180c675566f569ab7f9a68fc0f16bed2502a`
- **Author**: SpicyRabbitLeg
- **State**: OPEN (+1 / -2)
- **Verdict**: `merge-as-is`

## Context

Two-line fix for a one-line control-flow bug in
`core/config/markdown/loadMarkdownRules.ts`. The intent of
the loop was: "try `AGENTS.md` first, then `AGENT.md`, then
`CLAUDE.md`; stop after the first one that exists." The
existing code instead stopped on the **first iteration**
unconditionally, so if the workspace had no `AGENTS.md` but
did have `CLAUDE.md`, the `CLAUDE.md` was silently ignored.

## Design

`loadMarkdownRules.ts:42-47` — the `break` was outside the
`agentFileFound = true` branch, indented at the same level as
the `try`'s success path:

```ts
//  before
        agentFileFound = true;
      }

      break; // Use the first found agent file in this workspace
    } catch (e) {
```

That means **every** loop iteration broke after the `try`,
regardless of whether the file was actually found. The
`catch` block doesn't break (it just `continue`s implicitly),
but the `try`'s normal exit path always broke — including
when the inner `if (file does not exist)` early-returned past
the `agentFileFound = true` line.

The fix moves `break` inside the success branch:

```ts
//  after
        agentFileFound = true;
+       break; // Use the first found agent file in this workspace
      }
    } catch (e) {
```

Now the loop only stops when a file was actually loaded. The
fall-through to the next iteration on `if (file not found)`
behaves like the catch path. Total diff: `+1 / -2` (the blank
line between the closing brace and `break` also goes away).

## Risks

- **No new test.** The PR description leans on the cubic
  bot's auto-summary; a 2-line behaviour change with zero
  test coverage means the next `loadMarkdownRules.ts`
  refactor can re-break this trivially. A test asserting that
  with `[no AGENTS.md, no AGENT.md, present CLAUDE.md]` the
  loader returns the `CLAUDE.md` rules would lock the
  contract in 6-8 lines of jest. The repo already has
  `loadMarkdownRules.test.ts`-style fixtures elsewhere.
- **Iteration order assumption.** The fix is correct for
  whatever ordered list the loop iterates over, but the PR
  doesn't show that list. If the order is alphabetical
  (`AGENT.md`, `AGENTS.md`, `CLAUDE.md`), the original bug's
  symptom (CLAUDE never loaded) wouldn't match the title. If
  the order is `[AGENTS.md, AGENT.md, CLAUDE.md]` (as the PR
  title implies), the fix is exactly right. Worth a one-line
  comment in the file naming the precedence so it doesn't
  drift via a later "let's sort the array" cleanup.
- The bug existed because the author of the original loop
  almost certainly **meant** the post-conditional break and
  forgot to indent it inside the success branch. Adding a
  `// only stop on success — do not break on file-not-found`
  comment next to the new `break` would prevent a sympathetic
  reader from "fixing" it back the wrong way.

## Suggestions

- Add a unit test that constructs an in-memory IDE with only
  `CLAUDE.md` present and asserts the loader returns its
  rules with `alwaysApply: true`.
- Add a comment on the `break` explaining why it lives inside
  the `if (agentFileFound)` branch rather than at the
  `try`-suite level.
- Worth a one-line PR-body note linking the fallback order
  precedence (which file wins) so reviewers can confirm
  intent without reading the surrounding 60 lines.

## Verdict reasoning

`merge-as-is`. The fix is unambiguously correct, two lines,
and the risk of leaving the bug in place (silent ignore of
fallback agent files) is higher than the risk of merging
without a test. The test should land as a follow-up rather
than block this fix.

## What I learned

The "post-conditional break that should have been inside the
conditional" is one of the most common single-line bugs in
fallback loops, and the only reliable way to keep it from
recurring is to keep the **terminator** physically close to
the **success-condition mutation**. A pattern that's harder to
get wrong:

```ts
for (const file of FALLBACKS) {
  const rules = await tryLoad(file);
  if (rules) return rules;
}
return null;
```

The early `return` from inside the `if` is impossible to
misindent. The current shape uses a flag (`agentFileFound`)
and a manual `break`, which separates the "yes, I succeeded"
signal from the "stop looking" effect — and that gap is
exactly where the bug lived. A follow-up refactor to the
early-return shape would prevent the whole class.
