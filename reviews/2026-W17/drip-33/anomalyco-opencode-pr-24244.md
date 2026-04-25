# Review: anomalyco/opencode#24244 — Remove top-level wildcard-first sort in permission config

- **PR**: https://github.com/anomalyco/opencode/pull/24244
- **State**: OPEN
- **Author**: rekram1-node (Aiden Cline)
- **Range**: +18 / −34
- **Head SHA**: `bcb37c0d91d50d744a764619ad5154b972478241`
- **Base SHA**: `49894330d91c981e35be2002079442c8d0fe9836`
- **Verdict**: needs-discussion
- **Reviewer date**: 2026-04-25

## What the PR does

Removes the top-level key sort in
`packages/opencode/src/permission/index.ts:fromConfig`. The
previous code at lines ~288–303 sorted entries so wildcard
keys (`*`, `mcp_*`) came before specific keys (`bash`, `edit`).
Combined with `findLast` in `evaluate()`, this gave the
semantic "specific tool rules override the `*` fallback
regardless of JSON key order". This PR drops the sort and
leaves iteration in `Object.entries(permission)` insertion
order. Tests are inverted to match: the old assertion
`expect(wildcardFirst).toEqual(specificFirst)` becomes
`bash → allow` for one ordering and `bash → deny` for the
other.

## Observations

1. **This is a behavior reversal, not a bug fix.** The PR
   title says "fix: config ordering issue" but the existing
   sort is by design — see the deleted comment block at
   `permission/index.ts:289–292`: *"this gives the intuitive
   semantic 'specific tool rules override the `*` fallback'
   regardless of the user's JSON key order"*. There's no PR
   body explaining what's broken about that. Without
   motivation, this looks like a regression masquerading as
   a fix.
2. **The new "regression" test in `next.test.ts` is the
   smoking gun.** The added test:
   ```ts
   test("fromConfig - regression: trailing wildcard overrides earlier specific rule", () => {
     const ruleset = Permission.fromConfig({ bash: "ask", "*": "allow" })
     expect(Permission.evaluate("bash", "glab", ruleset).action).toBe("allow")
   })
   ```
   This test asserts that **a user who writes `bash: "ask"`
   followed by `*: "allow"` gets their `bash` rule
   silently overridden** by the trailing wildcard. That's
   exactly the footgun the original sort was guarding
   against. The PR ships this as a feature.
3. **JSON key order is not stable across config sources.**
   Users hand-write JSON, but tooling (formatters, schema
   migrations, `JSON.parse`/`stringify` round-trips through
   intermediaries, OS-level merging of layered configs)
   reorders keys. Making rule precedence depend on JSON key
   order means the same logical config can change behavior
   when a formatter runs. The original sort was the
   stability guarantee.
4. **The config.test.ts comment update reveals a contract
   change.** The deleted block said *"Rule precedence is NOT
   affected by this reordering"*. The new comment says
   *"permission evaluation uses last-match-wins
   precedence"*. That is a meaningful contract change for
   anyone whose config relies on the old guarantee. Needs
   a CHANGELOG entry and a migration note for users with
   existing `*: "deny"` + specific-allow configs.
5. **If the intent was to allow opt-in last-match-wins,
   the right shape is a config flag**, e.g.
   `permission.precedence: "wildcard-first" | "insertion-order"`,
   defaulting to `wildcard-first` to preserve current
   behavior. Flipping the default for everyone who has
   `*: "deny"` ahead of specific allows in their config is
   a breaking change that will silently lock them out of
   tools.
6. **Opportunity to capture the question:** is there a
   bug in the *original* sort (e.g., does it incorrectly
   sort `mcp_specific*` ahead of `mcp_*` due to substring
   matching)? If so, the fix is to refine the sort
   predicate, not to remove it. PR body is empty so we
   can't tell whether the author found an edge case in
   the sort or just disagrees with the policy.

## Verdict reasoning

This is a policy reversal that ships with no PR description,
no migration guide, no flag, and a "regression" test that
codifies the new footgun. It may be the right call (some
users want strict insertion-order semantics) but it should
not land silently. Recommend `needs-discussion`: maintainers
should weigh in on whether the original wildcard-first sort
was intentional design or accidental complexity, and
whether the change deserves a flag rather than a default
flip.

## What I learned

When a comment block says *"this gives the intuitive
semantic X regardless of the user's JSON key order"* and a
PR removes both the comment and the code that implemented
X, the burden of proof is on the PR to explain why X is no
longer the intuitive semantic. "Intuitiveness" claims in
config UX are reversible only with user-research evidence,
not by reading the code differently a year later.
