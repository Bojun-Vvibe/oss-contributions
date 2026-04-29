# block/goose #8878 — isolate GitHub recipe temp paths

- **PR:** https://github.com/block/goose/pull/8878
- **Title:** `fix: isolate GitHub recipe temp paths`
- **Author:** zuyua9
- **Head SHA:** 8350ad8e6af8d44117fab43983159389151b77b5
- **Files changed:** 1 (`crates/goose-cli/src/recipes/github_recipe.rs`),
  +52 / −3
- **Verdict:** `merge-after-nits`

## What it does

Two related bugs in one PR:

1. **Owner-shadowing collision** in `get_local_repo_path`
   (`github_recipe.rs:135-148`). Pre-change, any two repos sharing a
   `repo_name` (e.g. `alice/recipes` and `bob/recipes`) cloned into the
   same on-disk path under `local_repo_parent_path`, so the second
   `gh repo clone` either failed or — worse — fetched into a directory
   already holding a different owner's content.
2. **Path-traversal in temp dir** in `get_folder_from_github`
   (`github_recipe.rs:177` pre-change):
   `env::temp_dir().join(recipe_name)`. If `recipe_name` contained
   `../` segments, the joined path could escape `env::temp_dir()`. The
   new `temp_child_name()` sanitizer (lines 132-150) replaces `/` and
   `\` with `__` and any non-`[A-Za-z0-9.-_]` char with `_`, so
   `"../outside"` becomes `"..__outside"` and lands safely under the
   parent.

The clone itself is now also explicit:

```rust
Command::new("gh")
    .args(["repo", "clone", recipe_repo_full_name])
    .arg(&local_repo_path)
    .current_dir(local_repo_parent_path.clone())
```

— passing the destination path to `gh repo clone` as a positional arg
so the new owner-prefixed directory name is honored.

The two new tests at `github_recipe.rs:380-401` cover the headline
guarantees:

- `local_repo_path_includes_owner_to_avoid_collisions` — `owner-one/shared`
  and `owner-two/shared` resolve to distinct paths
  (`owner-one__shared`, `owner-two__shared`).
- `temp_child_name_keeps_recipe_downloads_under_temp_dir` —
  `temp_child_name("../outside") == "..__outside"` and the joined
  path satisfies `output_dir.starts_with(parent)`.

## What's good

- The two bugs are genuinely related — both are "untrusted segment
  used directly in a `Path::join`" — and fixing them together with one
  helper is the right call.
- `temp_child_name` whitelist (`[A-Za-z0-9.-_]`) is conservative but
  correct for filesystem-portable identifiers; avoids the
  unicode-normalization rabbit hole that a blacklist would hit.
- Empty-input handling: `if child.is_empty() { "_".to_string() }`
  (lines 145-149) prevents `recipe_name == ""` from collapsing into a
  zero-length path component that `join` would interpret as the parent
  itself.
- The owner-collision test is small, deterministic, and uses a
  literal parent path (`Path::new("goose-recipes")`) so it doesn't
  touch the real filesystem — safe to run under any cargo test setup.

## Nits / risks

- `temp_child_name("../outside") == "..__outside"` — that string
  starts with `..`. While the test asserts
  `output_dir.starts_with(parent)`, that's a string-prefix check on
  the `Path` components, which is satisfied because `..__outside` is
  one component, not `..`. Good. But worth adding a test for
  `temp_child_name("./.")`, `temp_child_name(".")`, and
  `temp_child_name("..")` — those resolve to `"_._"`, `"_"`, and
  `"_"` respectively under the current rules, which is safe but
  worth pinning down so a future "simplify the helper" refactor
  doesn't regress the dot-only edge cases.
- `temp_child_name` is also called inside `get_local_repo_path` for
  the **whole** `format!("{owner}/{repo_name}")`, which means an
  `owner` containing weird characters (`*`, `:`, etc.) becomes part
  of the directory name as `_`. That's correct, but two consecutive
  invalid chars collapse to two `_` rather than one, so
  `owner with spaces/repo` becomes `owner_with_spaces__repo` — fine
  for uniqueness but uglier than necessary. Optional cleanup: collapse
  consecutive `_` runs.
- The cloned repo path is now `owner__repo` rather than `repo`, which
  is a **behavior change** for users with existing
  `local_repo_parent_path` caches. On upgrade, those users will
  silently re-clone everything. Not catastrophic (repos are small), but
  worth a release-notes line so users with metered networks or large
  monorepo recipes aren't surprised.
- No test for the `gh repo clone <repo> <dest>` invocation itself;
  the new `.arg(&local_repo_path)` is exercised only at runtime. A
  small mock or trait-based test for `Command` argv would lock in the
  destination-passing behavior.

## Verdict rationale

`merge-after-nits`: real security/correctness fix, solid tests for the
core invariants, but the dot-only edge cases and the cache-invalidation
behavior change deserve a quick mention before merge. The optional
collapse-runs-of-underscore cleanup is a nice-to-have, not a blocker.
