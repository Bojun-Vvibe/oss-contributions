# PR #24652 — fix(config): make resource loading deterministic

- **Repo**: sst/opencode
- **PR**: #24652
- **Head SHA**: `daf72e46`
- **Author**: HaleTom (Tom Hale)
- **Size**: +150 / -95 across 8 files
- **URL**: https://github.com/sst/opencode/pull/24652
- **Verdict**: **merge-after-nits**

## Summary

Closes a non-determinism in config loading caused by brace-pattern
glob expansion: `Glob.scan("{agent,agents}/**/*.md", ...)` was
returning entries in filesystem-traversal order, which on at least one
platform does *not* deduplicate when both `agent/` and `agents/` exist
(the same file relative path under each subdir gets independently
listed) and the *order* of results is undefined across platforms.
Anywhere downstream that iterates and `result[name] = ...` ends up
with whichever entry happened to come last winning the key, which
varies between machines/CI runs.

The fix is two-pronged:
1. `Glob.scan` and `Glob.scanSync` now `return results.sort()`
   unconditionally at `packages/core/src/util/glob.ts:23,28`.
2. Every brace-glob in agent/command/mode/plugin/skill/tool loaders is
   replaced with an explicit `for (const subdir of ["X", "Xs"])` loop
   (or `["skill/**/SKILL.md", "skills/**/SKILL.md"]` array) so the
   iteration order across the two sibling dirs is deterministic *and*
   the loop body short-circuits per-subdir errors independently.

## Specific changes

- `packages/core/src/util/glob.ts:23,28` — `glob(...).sort()` and
  `globSync(...).sort()` shadow the underlying lib's order. Removes
  the `.sort()` calls scattered across the call sites in
  `test/util/glob.test.ts` (lines 17, 56, 96, 106, 138, 149) since
  the API now guarantees sorted output.
- `packages/opencode/src/config/agent.ts:111-145` — `load()` and
  `loadMode()` switched from `{agent,agents}/**/*.md` and
  `{mode,modes}/*.md` to nested `for (const subdir of ["agent","agents"])`
  / `["mode","modes"]` loops. Same shape applied at
  `packages/opencode/src/config/command.ts:28-67` for
  `{command,commands}/**/*.md`, `packages/opencode/src/config/plugin.ts:33-43`
  for `{plugin,plugins}/*.{ts,js}`, and `packages/opencode/src/tool/registry.ts:165-167`
  for `{tool,tools}/*.{js,ts}`.
- `packages/opencode/src/skill/index.ts:25,173-175` — special case:
  the constant is renamed `OPENCODE_SKILL_PATTERN` → `OPENCODE_SKILL_PATTERNS`
  (an array of two patterns) and the discovery scan loop iterates
  over them. Note that `EXTERNAL_SKILL_PATTERN` (`.claude` / `.agents`
  external skills) keeps the singleton `skills/**/SKILL.md` form,
  which is correct because those tools never wrote a `skill/`
  (singular) variant.
- New test `test/util/glob.test.ts:118-128` writes `c.txt`, `a.txt`,
  `b.txt` in a non-alphabetical order and asserts the scan returns
  `["a.txt","b.txt","c.txt"]` — pins the new sorted-output contract.
- Bundled change at `test/shell/shell.test.ts:30-50` — adds five new
  tests for `Shell.ps`, `Shell.args` flags-for-bash/zsh/nu/fish/cmd.
  This is *not* related to the deterministic-resource-loading fix.

## Risks

- **Bundled scope**: the shell.test.ts additions (~30 lines of test
  code for `Shell.ps` / `Shell.args` coverage) have no relationship to
  glob sorting or config loaders. They're harmless individually but
  should ideally be a sibling PR — a maintainer reviewing "make
  resource loading deterministic" doesn't expect a shell-args test
  expansion in the diff. Won't block merge but worth noting.
- **Performance**: every `Glob.scan` call now pays an O(n log n) sort
  on the result list. For the file-tree scans this code does (typically
  <100 entries per `Glob.scan` call site), the cost is in the noise.
  If there's a hot path that scans thousands of files via this API,
  the sort cost would surface — none of the call sites in the diff
  look like that shape but worth a quick mental check.
- **Plugin order dependency**: `config/plugin.ts:35-41` previously
  loaded `{plugin,plugins}/*.{ts,js}` in unspecified order; the new
  code loads `plugin/` then `plugins/` and within each, sorted by name.
  If any installed plugin relied on a specific load-order (e.g.
  registers a hook expecting another plugin to have already
  registered), that ordering changes. Probably not the case in
  practice (plugins should be independent), but a one-line release
  note would help anyone hitting it.
- **No deduplication**: if a user has both `agent/foo.md` and
  `agents/foo.md` on disk, the new code now deterministically
  *overwrites* `result["foo"]` with whichever subdir comes second
  (always `agents/`). That's a sensible "agents wins" rule but it's
  implicit — a one-line comment naming the precedence at
  `config/agent.ts:111` would save the next reader a `git blame`.

## Verdict

**merge-after-nits**: the core fix (sort + explicit subdir iteration)
is correct, the shape generalizes uniformly across all 6 loaders, and
the new test pins the contract. Two real nits: split the shell.test.ts
additions to a sibling PR, and add a one-line "agents wins over agent"
precedence comment at `config/agent.ts:111`. The "no dedup" surprise
is the kind of thing that bites someone in 6 months when they realize
their `agent/foo.md` is being silently shadowed.

## What I learned

When a sort is added to make output deterministic, the *right* layer
to add it is the lowest-level API (here `Glob.scan` / `Glob.scanSync`)
not the consumers. This PR does both — pushes the sort into the API
*and* removes the redundant `.sort()` calls scattered in the test
file. That's the cleanest end state because future call sites get
determinism for free without having to remember to `.sort()`. The
"replace brace-glob with two-iteration loop" pattern is also worth
internalizing: brace expansion is convenient syntax but it
deduplicates inconsistently across glob libraries (some libs collapse
`{a,a}` to `a`, others don't), and it makes the iteration order of
the two arms implementation-defined. An explicit `for (const subdir
of [a, b])` loop sidesteps both ambiguities and the loop body can
short-circuit per-subdir errors independently — which is invisible
gain until the day a malformed file in `agents/` blows up the whole
load pass that should have at least returned the entries in `agent/`.
