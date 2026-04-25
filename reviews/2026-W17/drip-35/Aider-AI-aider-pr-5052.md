# Aider-AI/aider #5052 — Add bash/shell repomap support

- **Repo**: Aider-AI/aider
- **PR**: [#5052](https://github.com/Aider-AI/aider/pull/5052)
- **Head SHA**: `fd177080f19d59c1b6a71002ae64d19d950be84f`
- **Author**: Kcstring
- **State**: OPEN (+38 / -0)
- **Verdict**: `merge-after-nits`

## Context

Aider's repo map relies on per-language tree-sitter tag queries under
`aider/queries/tree-sitter-language-pack/`. Bash was missing, so `.sh`
/ `.bash` / `.zsh` files contributed nothing to the map even though
they often carry meaningful function definitions in real projects
(devops repos, monorepo helpers, install scripts).

## Design

The new query at
`aider/queries/tree-sitter-language-pack/bash-tags.scm:1-6` is minimal
and correct:

```scheme
(function_definition
  name: (word) @name.definition.function) @definition.function
```

Cross-checked against the upstream `tree-sitter-bash` grammar: the
`function_definition` node is the canonical container for *both*
declaration styles (POSIX `name() { … }` and keyword `function name
{ … }`), and `name: (word)` is the field on it. So the same one query
covers both styles the PR description mentions, which the fixture at
`tests/fixtures/languages/bash/test.sh:5-10` exercises.

The test wiring at `tests/basic/test_repomap.py:287-288`:

```python
def test_language_bash(self):
    self._test_language_repo_map("bash", "sh", "greet")
```

mirrors the existing pattern (`test_language_c`, etc.) exactly. The
helper `_test_language_repo_map` already knows how to load the
fixture, run the repo map, and assert the symbol shows up.

## Risks / Nits

1. **Aliased extensions not covered by the test.** The PR description
   says `.sh`, `.bash`, `.zsh` work, but the fixture only exercises
   `.sh`. Whether `.bash` / `.zsh` actually route through the bash
   parser depends on the language-detection table elsewhere in aider
   (likely `aider/repomap.py`'s extension map). This PR doesn't touch
   that table — verify it already maps those extensions, otherwise the
   description overpromises. Worth a one-line check; if the mapping is
   missing, that's a small follow-up.

2. **`function name { … }` (no parens) not in the fixture.** The query
   captures it (the grammar treats it the same), but the fixture only
   shows `function deploy()` (with parens, per the diff). Adding a
   third function `function nodryun { echo no; }` to the fixture would
   lock that case down too. Optional but cheap.

3. **Heredocs / sourced files.** The query intentionally only captures
   `function_definition`. Variables, aliases, and `source`-d functions
   won't appear in the map. That's the right scope for a repo map —
   matches what other languages do (Python captures `def` / `class`,
   not module-level constants).

4. **No `@reference` captures.** Other tag files in this directory
   capture both definitions and call-sites with
   `@reference.call`. Bash repomap won't show "X is called from Y"
   relationships. Not blocking — most of the existing language packs
   are def-only — but a follow-up could add
   `(command name: (command_name) @name.reference.call) @reference.call`
   to enable cross-file linkage.

## Verdict rationale

`merge-after-nits`. The query is correct, the test runs the fixture
end-to-end, and the patch is strictly additive (new file + one
test method + one fixture). The only thing I'd ask before merge is a
quick check that the extension map actually routes `.bash` / `.zsh` to
this parser — if not, either narrow the PR description or add the map
entry in the same change.

## What I learned

Adding a new language to a tree-sitter-based repo map is a satisfyingly
small surface — one `.scm` query, one fixture, one test. The hard part
is knowing the grammar's node names; once you do, the capture pattern
is two lines. The fact that `function_definition` covers both bash
declaration styles is a nice property of the upstream grammar that this
patch quietly inherits.
