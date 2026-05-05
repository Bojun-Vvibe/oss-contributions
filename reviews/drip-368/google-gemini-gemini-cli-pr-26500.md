# google-gemini/gemini-cli PR #26500 — fix(core): enable ripgrep to search hidden directories by default

- Head SHA: `cf86f345767b37c94b14d995f9d6d64a2a74816c`
- Author: `cynthialong0-0`
- Size: +23 / -1
- Verdict: **needs-discussion**

## Summary

Single-character behavior change in
`packages/core/src/tools/ripGrep.ts:405`: the `rgArgs` array used to
spawn `rg` is changed from `['--json']` to `['--json', '--hidden']`,
making **every** invocation of the `grep_search` tool descend into
dotfiles and dot-directories (`.git/`, `.env`, `.venv/`,
`node_modules/.cache/`, `.aws/`, `.ssh/`, `.npmrc`, …) by default,
with no opt-out parameter and no per-call control.

## What the diff actually does

- `packages/core/src/tools/ripGrep.ts:405` — flips the default
  ripgrep CLI arg set to include `--hidden`, which per `rg --help`
  causes ripgrep to "search hidden files and directories" that are
  otherwise skipped (`.foo` paths). This is *separate* from
  `--no-ignore` — `.gitignore` rules are still honored, so the
  expansion is bounded by what's both hidden and not gitignored.
- `packages/core/src/tools/ripGrep.test.ts:1333,1781` — adds
  `expect(spawnArgs).toContain('--hidden')` to two existing spawn
  arg-shape tests, pinning the new default.
- `evals/grep_search_functionality.eval.ts:182-200` — new
  `USUALLY_PASSES` behavioral eval that drops a fixture file at
  `.hidden_dir/system.md` and asserts the model can find content
  in it via `grep_search`.

## Why needs-discussion

This is a security-sensitive default flip dressed up as a 1-line
QoL improvement. Three concrete concerns the PR doesn't address:

1. **Credential-file exposure.** The change makes it the default
   that any prompt-injected or model-confused `grep_search` call
   for a string like `password`, `token`, `key`, `BEGIN PRIVATE`,
   `aws_access_key_id`, `bearer`, etc. now traverses `.env`,
   `.envrc`, `.aws/credentials`, `.npmrc` (with `_authToken`),
   `.ssh/config`, `.git-credentials`, and similar dotfiles in the
   workspace. Pre-change, those files were invisible to the tool.
   The eval added at `evals/grep_search_functionality.eval.ts:182`
   *actively rewards* the model for finding hidden-dir content,
   which trains the loop to lean into this behavior.

2. **`.git/` traversal is a real performance regression too.**
   Even on repos where `.gitignore` excludes nothing extra,
   `--hidden` makes ripgrep walk `.git/objects/`,
   `.git/logs/refs/`, etc. on every invocation. ripgrep is fast
   but `.git/` on a multi-GB repo is non-trivial (loose objects,
   pack indexes). `.gitignore` does *not* exclude `.git/`
   itself — it's filtered by ripgrep's separate "skip VCS dirs"
   logic which `--hidden` does not bypass on its own, but the
   PR's own assertion that hidden dirs are now searched suggests
   the author hasn't characterized which hidden paths actually
   make it through. A targeted test against a fixture with
   `.git/` should be added to confirm.

3. **Loss of opt-in.** ripgrep itself ships `-.`/`--hidden` as
   opt-in for exactly these reasons. The right shape for this
   change is a per-tool-call boolean parameter
   (`include_hidden: boolean = false`) plumbed through the
   tool's input schema, so the model can ask for hidden paths
   when it has a reason and the user can see the explicit
   opt-in in the tool-call trace. Making it the unconditional
   default removes that signal entirely — no audit log will
   show "the model went into `.env`" because every call goes
   into `.env`.

## What I'd want before this lands

- **Either**: revert to opt-in with a new `include_hidden` param
  (default `false`), update the schema description, route the
  new param into the `--hidden` flag conditionally, and keep the
  new eval but mark it as exercising the explicit opt-in path.
- **Or**: keep the default flip but add an explicit denylist
  for dotfile credential paths (`.env*`, `.aws/`, `.ssh/`,
  `.npmrc`, `.netrc`, `.git-credentials`, `.pgpass`,
  `.docker/config.json`, …) baked into the default `rgArgs`
  via `--glob '!.env*'` etc., document the denylist in the tool
  description, and add an explicit test asserting the denylist
  is honored.

The substantive concern is not "hidden search is wrong" — it's
"hidden search by default with no opt-out and no credential-file
guardrail" is a meaningful expansion of the agent's read surface
that warrants an explicit design call from the gemini-cli
maintainers. The 1-line + 22-line-test framing undersells the
change.

## Optional nits (also worth fixing)

- `evals/grep_search_functionality.eval.ts:194` — regex
  `/\\.hidden_dir[\\\\/]system\\.md/` has too many backslashes
  for the character-class case. The intent is "match `\` or `/`
  as the path separator", which is `[\\/]` in JS regex source,
  i.e. in a TS string literal `[\\\\/]` — but the leading
  `\\.hidden_dir` should be `\\.hidden_dir` (one literal `\.`),
  which the current source produces correctly. Worth a manual
  sanity-check against the actual eval output to confirm this
  matches `./.hidden_dir/system.md` and `.\.hidden_dir\system.md`.
- The eval is `USUALLY_PASSES` rather than `MUST_PASS`, which
  means a regression in the default would not flip CI red. If
  the maintainers decide to keep the default-on behavior, this
  should be promoted to `MUST_PASS`.
