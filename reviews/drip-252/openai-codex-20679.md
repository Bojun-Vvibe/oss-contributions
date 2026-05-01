# openai/codex #20679 — Detect implicit skill reads with parsed commands

- **Repo:** openai/codex
- **PR:** https://github.com/openai/codex/pull/20679
- **HEAD SHA:** `26c058dd2ec91abd83b5eacf0ed0ad92600aae80`
- **Author:** alexsong-oai
- **Verdict:** `merge-after-nits`

## What the diff does

Replaces the hand-rolled "is this a file-reading command?" detector
in `core-skills` with a delegation to the canonical shell-command
parser already used elsewhere in the codebase, fixing the case where
a `cat SKILL.md | head` pipeline (or `nl SKILL.md`) was not
recognized as a skill-doc read and therefore didn't trigger implicit
skill loading.

Three changes in `codex-rs/core-skills/`:

1. `Cargo.toml:23` — adds `codex-shell-command` dep.
2. `src/invocation_utils.rs:103-127` —
   `detect_skill_doc_read` now iterates the `parse_command_impl`
   output, picks `ParsedCommand::Read { path, .. }` arms, and
   resolves each path against the skill index, replacing the prior
   `READERS` allow-list (`["cat","sed","head","tail","less","more","bat","awk"]`)
   plus the leading-`-` flag-skip heuristic. New helper
   `tokens_with_command_basename` at `:120-128` normalizes the
   first token (basename, lowercase) before handing the slice to
   the parser so `/usr/bin/cat` and `cat` both reach the same
   parsed shape.
3. `src/invocation_utils_tests.rs:78-100` — new unit test
   `skill_doc_read_detection_uses_parsed_read_commands` pinning
   `nl /tmp/skill-test/SKILL.md`.
4. `core/tests/suite/skills.rs:154-194` — integration test
   `parsed_skill_doc_reads_detect_loaded_repo_skill` writes a repo
   skill, asserts `nl <path>` detects it and `echo <path>` does
   not (negative arm proves the parser-driven gate isn't
   over-fire-ing on every command that mentions a path).

## Why the change is right

The prior implementation had two compounding problems:

1. **Reader allow-list was fixed-set, not extensible.** A user who
   ran `nl SKILL.md`, `xxd SKILL.md`, `pv SKILL.md`, or any of a
   dozen other reader-shaped commands would silently fail implicit
   skill loading. `parse_command_impl` already encodes the
   read-shape detection elsewhere in codex (used for the agent's
   own command parsing), so the dup-and-drift was real.

2. **Pipeline blindness.** `cat SKILL.md | head` decomposes into a
   pipeline of two commands at the shell parser; the prior
   first-token check only saw `cat` and stopped at the first
   non-flag arg, which happened to work for `cat SKILL.md`
   directly but failed on any pipeline form. The new code iterates
   over *all* `ParsedCommand::Read` arms produced by the parser,
   so multi-stage pipelines that contain a read of a skill doc
   are detected at every read site.

The negative-arm test (`echo <path>` returns None) is the load-
bearing assurance that the parser-driven detection isn't a
super-set of "any token that resolves to a skill path" — `echo`
isn't a `Read`-shaped parsed command so it's correctly excluded.

`tokens_with_command_basename` at `:120-128` is the right place to
normalize — `/usr/bin/cat` should reach the parser as `cat`
because that's what `parse_command_impl`'s `Read` detection keys
on, but the rest of the argv must pass through unchanged because
that's where the path-of-interest lives.

## Nits (non-blocking)

1. **Behavior change for flag args.** The old code had
   `if token.starts_with('-') { continue; }` to skip flags before
   path resolution; the new code delegates this to `parse_command_impl`.
   For a command like `cat -n SKILL.md`, the parser presumably
   handles the `-n` correctly, but for `tail --lines=5 SKILL.md`
   or `bat --paging=never SKILL.md` it depends entirely on whether
   `parse_command_impl` knows about each tool's flag syntax. A
   regression test for `cat -n <path>` and `tail --lines=5 <path>`
   would lock the most common cases.

2. **`parse_command_impl` failure modes** are silent. If the
   parser fails (returns empty, panics on weird quoting,
   semicolon-chained commands like `cd /tmp && cat SKILL.md`),
   the iterator-over-empty produces no `Read` arms and detection
   silently misses the skill. The old code at least had
   well-understood deterministic failure ("unknown reader →
   no detection"). A trace-level log or a passthrough test for
   `cd /tmp && cat SKILL.md` would clarify which shapes are
   intentionally out of scope.

3. **`canonicalize_if_exists` cost in pipeline scans.** The new
   code calls `canonicalize_if_exists(&workdir.join(path))` once
   per `Read` arm. For long pipelines (rare but possible:
   `cat a.md b.md c.md | grep ... | head`), this is N syscalls
   per implicit-skill detection. Caching the canonical form per
   `outcome.implicit_skills_by_doc_path` lookup, or short-
   circuiting on the first match, would bound the cost. Not
   load-bearing — just worth noting if implicit-detection ever
   shows up in turn-startup-latency profiling.

4. **Implicit drop of `awk`.** The old `READERS` list included
   `awk`. `awk -f script.awk SKILL.md` reads SKILL.md, but
   `parse_command_impl` may or may not classify `awk` invocations
   as `Read` depending on its argument shape (awk programs can
   take input from files or stdin). Worth a quick check that
   `awk '{print}' SKILL.md` is still detected; if not, this PR
   is a behavior-narrowing change for awk users that should be
   called out in the description.

5. **Test naming asymmetry.** The new unit test is
   `skill_doc_read_detection_uses_parsed_read_commands` (positive)
   but there's no negative-arm unit test — the negative coverage
   only exists in the integration test at
   `core/tests/suite/skills.rs:191`. A unit-level
   `skill_doc_read_detection_skips_non_read_commands` asserting
   `echo SKILL.md` returns `None` would localize the negative
   contract to the same file as the positive one.

## Verdict rationale

Right delegation to the canonical parser, eliminating dup-and-drift
between `core-skills` and the rest of the codebase. Pipeline support
is a real correctness improvement. Locking tests in both unit and
integration layers, including the negative arm. The behavior-narrowing
risk for `awk` users + flag-syntax edge cases (`tail --lines=5`) are
the only real review items.

`merge-after-nits`
