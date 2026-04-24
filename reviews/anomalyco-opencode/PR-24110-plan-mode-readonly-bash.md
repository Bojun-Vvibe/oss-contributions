# PR-24110 — fix(opencode): enforce read-only bash permissions in plan mode

[sst/opencode#24110](https://github.com/sst/opencode/pull/24110)

## Context

Plan-mode previously inherited the default broad bash permissions and
only denied edit-class tools, so a model in plan mode could still
execute `git reset --hard`, `npm install`, `chmod -R 777`, etc. This
PR adds two things in `packages/opencode/src/agent/agent.ts`:

1. A new `permissions(agent, session)` helper — for the `plan` agent
   it inverts the merge order so `Permission.merge(session, agent.
   permission)` is used (agent rules win); for everything else it
   keeps `Permission.merge(agent.permission, session)`.
2. A built-in plan-mode bash deny-list with a small allowlist of
   read-only inspection commands (`git status *`, `git log *`,
   `git diff *`, `git show *`, `git branch`, `git stash list *`,
   `ls *`, `cat *`, `grep *`, `rg *`, `find *`, `wc *`, `head *`,
   `tail *`).

Call sites in `session/llm.ts` and `session/prompt.ts` are updated to
use `Agent.permissions(...)` instead of calling `Permission.merge`
directly. Tests in `test/agent/agent.test.ts` cover both the allow
list, an explicit deny list (`git push -f`, `rm -rf`, `npm install`,
inline `python -c open(...)`), and the override semantics: a
user-configured `npm install *: allow` is overridden by the plan-mode
`*: deny`.

## Strengths

- The merge-order flip is the right surgical fix. Rather than
  inventing a new "locked rule" concept, it just changes which side
  of `Permission.merge` wins for `plan`. Centralizing this in
  `Agent.permissions` means the four call sites can't drift.
- Test coverage is genuinely useful: `python -c 'open("x", "w").write
  ("y")'` is denied via the catch-all `*` even though it's not in any
  explicit list — that exercises the precedence logic, not just the
  allowlist.
- Allowlist patterns terminate at the command boundary (`git status
  *`, `cat *`), so chained commands like `git status; rm -rf` would
  fail the wildcard match cleanly.

## Concerns / risks

- **Branch on agent name, not capability.** The override hinges on
  `if (agent.name === "plan")`. A user who clones the plan agent and
  renames it (`my-plan`, `architect-plan`) loses the read-only
  guarantee silently. A capability marker on `Info` (e.g.
  `permissionMode: "agent-wins"` or `readonly: true`) would be more
  resilient than a string compare.
- **Allowlist is happy-path only.** Many useful read-only tools are
  absent: `awk`, `sed -n`, `jq`, `bat`, `tree`, `du`, `df`, `stat`,
  `file`, `which`, `pwd`, `env`, `pwd`, `git remote -v`, `git log
  --graph`, `git rev-parse`, `git diff --stat`. Users will hit deny
  walls on routine inspection. Worth either documenting the
  intentional minimalism or expanding to cover the standard
  inspection toolbox.
- **Globs match commands, not arguments.** `cat *` matches `cat
  /etc/shadow` just as happily as `cat README.md`. Plan mode is
  intended to be safe-by-default, but allowing arbitrary `cat` and
  `grep` lets an instruction-injected model exfiltrate secrets via
  tool output (the model sees the file content, which then ends up
  in the next turn's context). For a true safety floor, `cat` /
  `grep` need at minimum a "must be inside cwd" check — the current
  approach silently widens the read surface to the whole filesystem.
- **`git stash list *` is the only `git stash` subcommand allowed,**
  which is the right choice (stash apply/pop/drop mutate). But
  `git stash` with no args defaults to `git stash push`, so a model
  that types `git stash` (matching nothing in the allowlist) gets
  cleanly denied — verify the allowlist matcher requires the trailing
  ` ` boundary so `git stash listfoo` isn't accepted.
- **Override test only checks one direction.** The test asserts that
  a session-level `pattern: "*", action: "deny"` *cannot* downgrade
  the plan-mode `git status: allow` (good — agent rules win). But
  the symmetric test — that a session-level `bash: { "rm -rf *":
  "allow" }` cannot escape plan mode — is not present. Worth adding
  to lock the policy.

## Verdict

**Approve with caveats.** The merge-order inversion is correct and
the tests exercise the right primitives. Before/after merge: replace
the agent-name string compare with a typed capability flag, expand
the allowlist to cover standard inspection commands, and add a test
that explicitly tries to escape plan mode via session-level allow
rules. Document that `cat`/`grep` on this allowlist do not constrain
file paths — that's a deliberate trade-off the reader should see.
