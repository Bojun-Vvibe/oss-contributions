# QwenLM/qwen-code#3570 — feat(core): add simplify bundled skill

- **Head**: `17c5f7cb3494be005555ba7746bb89270a7d2374`
- **Size**: +182/-8 across 3 files
- **Verdict**: `merge-as-is`

## Context

Adds a new bundled `/simplify` skill to the qwen-code CLI. The skill
defines a structured cleanup workflow: identify the review scope (git
diff), launch three parallel agent passes (code reuse, code quality,
efficiency), aggregate findings, then directly apply low-risk cleanup
edits with verification. Touches the bundled-skill loader test and the
skill-manager test to confirm the new skill is discovered and registered
the same way the existing `review` bundled skill is.

## Design analysis

### The skill content: `packages/core/src/skills/bundled/simplify/SKILL.md:1-123`

The frontmatter declares the right surface:

```yaml
name: simplify
description: Review recent code changes for reuse, code quality, and efficiency,
  then directly apply straightforward cleanup improvements...
allowedTools:
  - agent
  - run_shell_command
  - grep_search
  - read_file
  - write_file
  - edit
  - glob
```

The `allowedTools` list is the right minimum: `agent` for the parallel
review passes, `read_file`/`grep_search`/`glob` for inspection, `edit`/`write_file`
for the apply-cleanup step, and `run_shell_command` for the `git diff`
identification step. Notably absent: `delete_file`, network tools, and
shell-execution-of-tests beyond what `run_shell_command` already
permits. That's a sensible scope — a "simplify" pass shouldn't be
deleting files or running flaky network-dependent tests.

### The workflow shape

Steps 1–7 (review scope → 3 parallel passes → aggregate → confirm →
apply low-risk → verify → report) is a coherent contract. Two design
choices worth flagging:

1. **Parallel passes via the `agent` tool launched in a single response**
   (Step 2: "launch all review passes in a single response so they run
   concurrently"). This is the right pattern — sequential review passes
   would triple the wall-clock time without quality gain, since the
   three perspectives are independent. The instruction to "not paste
   the full diff into the prompt" and instead "tell each pass to read
   the diff itself" is also correct: keeps prompts cheap and lets each
   sub-agent decide its own context window.

2. **Apply low-risk improvements directly, defer high-risk to user**
   (Step 5/6). This is the user-trust-preserving default. Defining
   "low-risk" via the explicit list (renames, dead-code removal, simple
   inlining, formatter-equivalent cleanups) is more useful than a vague
   "use your judgment" instruction.

### Loader integration: `BundledSkillLoader.test.ts:142-162`

```ts
it('should load simplify bundled skill like other slash commands', async () => {
  const skills = [
    makeSkill({
      name: 'simplify',
      description: 'Simplify recent changes',
      filePath: '/bundled/simplify/SKILL.md',
      body: 'Simplify body',
    }),
  ];
  mockSkillManager.listSkills.mockResolvedValue(skills);
  const loader = new BundledSkillLoader(mockConfig);
  const commands = await loader.loadCommands(signal);
  expect(commands).toHaveLength(1);
  expect(commands[0].name).toBe('simplify');
  expect(commands[0].description).toBe('Simplify recent changes');
  expect(commands[0].kind).toBe(CommandKind.SKILL);
});
```

Mirrors the existing review-skill test pattern. Pins the contract:
loader produces a single `CommandKind.SKILL` command named `simplify`
with the declared description. Good.

### Skill-manager integration: `skill-manager.test.ts` (truncated diff)

The visible diff hunks reshape `setupReviewSkillMocks` into a
file-path-conditional `fs.readFile` mock and a content-conditional
`mockParseYaml`, so a single test setup can produce both `review` and
`simplify` skills. Then asserts:

```ts
expect(skills.some((s) => s.name === 'simplify')).toBe(true);
const simplifySkill = skills.find((s) => s.name === 'simplify');
expect(simplifySkill!.level).toBe('bundled');
```

That's the right level pin — confirming the skill is discovered as a
*bundled* skill (not project-level or user-level), so it gets the same
loader treatment as `review`.

The mock refactor itself (from a single static return value to a
path/content-conditional implementation) is fine but does increase the
fragility of the test setup: any future skill added to `BundledSkillLoader`
will need a similar conditional branch. Worth flagging as a refactor
target if a third bundled skill arrives.

## Concerns / nits

1. **The skill file is 123 lines of prompt content.** Most of it is
   well-structured (numbered steps, named passes, explicit tool list).
   No defensive checks needed for prompt-content correctness at code
   review time — that's a runtime/eval concern. But if the project has
   a "skills must be ≤N lines" convention, worth checking; some skill
   frameworks cap to keep prompt cost predictable.
2. **Step 2's "launch all review passes in a single response"
   instruction** depends on the host CLI faithfully dispatching the
   parallel `agent` calls. If the host serializes them, the wall-clock
   benefit is lost. Probably out of scope for the skill itself, but
   worth a note in the skill's own troubleshooting section.
3. **No project-level customization hook.** Users with strong opinions
   on "what counts as low-risk" (e.g. teams that disallow auto-renames
   in shared API surfaces) can't easily override the rules without
   shadowing the entire skill. The 5-tier skill resolution chain
   handles this naturally if a user creates `~/.qwen/skills/simplify/SKILL.md`,
   but a one-line note in the skill body pointing to that override
   path would help.
4. **`mockParseYaml.mockImplementation((yamlString: string) => ...)`
   conditioning on `yamlString.includes('name: simplify')`** is brittle:
   if a future skill body legitimately contains the substring `name: simplify`
   (e.g. a code example in another skill's body), the mock will misroute.
   A cleaner alternative is to switch on the `filePath` argument from
   the prior `fs.readFile` mock. Minor test-cleanliness nit.

## Verdict reasoning

Pure additive feature: new bundled skill file, two test additions, no
behavioral change to existing skills. Both tests pin the right contracts
(loader produces a `SKILL` command; manager discovers at `bundled` level).
The skill content itself is well-structured and uses the right `allowedTools`
scope. Nits are all cosmetic (skill prompt length, mock-test brittleness,
override discoverability). This is a `merge-as-is` for me.

## What I learned

"Three parallel agent review passes" as a single skill primitive is a
nice formalization of a pattern that ad-hoc agents already do less
reliably. The explicit naming of the three perspectives (reuse / quality
/ efficiency) gives reviewers a checklist to verify the skill's output
is actually structured the right way, rather than collapsing all three
passes into "general code review". The same pattern would generalize
well to `/audit` (security / privacy / compliance) and `/perf-review`
(allocations / I/O / parallelism). The skill's choice to apply low-risk
edits directly rather than just commenting is also a useful model:
"review skills should optionally also fix" rather than "review skills
only ever produce text" is the higher-utility design.
