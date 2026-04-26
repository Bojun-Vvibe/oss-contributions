# Aider-AI/aider PR #5011 — Add --max-reflections CLI option

- Repo: Aider-AI/aider
- PR: #5011
- Head SHA: `3e7a1dfb`
- Author: @Discotech17
- Diff: 3 files — `aider/args.py`, `aider/coders/base_coder.py`, `aider/main.py`
- Closes: #3865

## What changed

Promotes the previously-hardcoded `max_reflections = 3` class attribute on `Coder` to a configurable parameter:

1. **`aider/args.py:268-274`** — adds `--max-reflections` argument (`type=int`, `default=3`), so it's also reachable via `AIDER_MAX_REFLECTIONS` env var and `max-reflections: 5` in `.aider.conf.yml` (configargparse handles those automatically).
2. **`aider/coders/base_coder.py:101`** — removes the class-level `max_reflections = 3`. Adds `max_reflections=3` to `__init__` signature (line 340) and `self.max_reflections = max_reflections` in the body (line 354).
3. **`aider/main.py:1007`** — passes `max_reflections=args.max_reflections` into the `Coder.create(...)` kwargs block.

## Specific observations

- The class-attribute → instance-attribute migration is clean and the default `3` is preserved at every layer (argparse default, `__init__` default, `Coder.create` passthrough). No behavior change for any existing user, which matches the PR description's "default remains 3" claim. Verified by reading `base_coder.py:337-354` — the new `__init__` parameter has `default=3` so subclasses or in-process callers that don't pass it still get the old behavior.
- `Coder.create` is the indirection layer between `main.py` and `Coder.__init__` — confirms it forwards `**kwargs` (it already passes `auto_copy_context`, `auto_accept_architect`, `add_gitignore_files`, etc. the same way). Fine.
- One minor: `--max-reflections=0` is now syntactically valid (no argparse `choices` or `min` constraint). With `max_reflections=0` the reflection loop short-circuits, which is actually a useful "single-pass strict" mode and probably not worth blocking. But `--max-reflections=-1` is also accepted and would loop forever (or never, depending on how `num_reflections >= max_reflections` is evaluated — `0 >= -1` is true so it'd never reflect). Worth a `type=positive_int` helper or a `if args.max_reflections < 0: parser.error(...)` to fail loudly rather than silently degrade.
- Two callers also construct `Coder` (subclasses + tests). A grep for `Coder(` would tell us if any of them set `max_reflections` directly as a class attribute override — if so, the migration to instance attribute might break them. Worth one `grep -rn "max_reflections" aider/ tests/` to confirm.
- No new test. A trivial test in `tests/basic/test_main.py` that asserts `--max-reflections 7` propagates through `Coder.create` would lock the wiring in.

## Verdict

`merge-after-nits`

## Rationale

Clean, well-scoped feature. Default-preserving by construction. Two small asks: validate `args.max_reflections >= 0` to prevent silent footguns, and add a one-line test that asserts the CLI flag reaches `Coder.max_reflections`.
