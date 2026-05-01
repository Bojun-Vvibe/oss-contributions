# Review: block/goose #8945 — remove artifacts dir handling

- URL: https://github.com/block/goose/pull/8945
- Head SHA: `d6c0e2dd8df2d7bf4906ad0343ebc468d288d3ce`
- Files: 22 (all under `ui/goose2/`)
- Size: +95/-402 (net −307 LOC)

## Summary of intended change

Removes the goose2 desktop app's special-case handling of an `artifacts/`
directory inside each project's storage. Per the PR body: "Removes the
special treatment of /artifacts, leaving that up to the prompting." The
change spans Tauri capabilities, the Rust path resolver, the projects
command surface, the React `ArtifactPolicyContext`, and supporting
i18n/test files.

Concrete deletions / changes:

1. **Tauri capability narrowing** at
   `ui/goose2/src-tauri/capabilities/default.json:19-25` and
   regenerated `gen/schemas/capabilities.json`: drops the
   `$HOME/.goose/artifacts/**` path from the `opener:allow-open-path`
   allowlist. The broader `$HOME/.goose/**` glob still covers the path
   if the user opts in via prompting, but the dedicated artifacts entry
   is gone.
2. **`projects.rs`** at `ui/goose2/src-tauri/src/commands/projects.rs`:
   - Deletes `fn project_artifacts_dir(...)` at `:121-124` (was
     `project_dir.join("artifacts").to_string_lossy()`).
   - Drops the `pub artifacts_dir: String` field from `ProjectInfo`
     at `:170`.
   - `project_info_from_stored` signature simplifies from
     `(project_dir: &Path, stored: StoredProjectInfo)` to just
     `(stored: StoredProjectInfo)` at `:173`, since the path was only
     used to compute the deleted field. Five call sites updated:
     `list_projects:217`, `list_archived_projects:250`,
     `create_project:323`, `update_project:368`, `get_project:394-395`
     (the last switches to `let (_, info) = ...` correctly dropping
     the unused dir).
   - Test deletion at `:472-479`:
     `builds_project_artifacts_dir_inside_project_storage`. Correct —
     the function under test is gone.
3. **`goose_serve.rs:135`**: `default_serve_working_dir()` simplifies
   from `dirs::home_dir().join(".goose").join("artifacts")` to
   `dirs::home_dir()`. **This is the load-bearing semantic change**:
   any goose2 install that didn't have an explicit `working_dir`
   override on the spawned server will now have its working directory
   set to `$HOME` rather than `$HOME/.goose/artifacts`. Tools the
   server invokes (file writes, shell commands run via the agent) that
   used relative paths will now resolve against `$HOME`. **Worth
   confirming** in PR body that all in-tree spawn callers explicitly
   pass a working_dir (otherwise this changes behavior for every
   `goose serve` started by goose2).
4. **`path_resolver.rs:64-95`**: tests rewritten to use
   `["src"]` / `["~"]` / `["~/Documents"]` / `["~\\Documents"]`
   instead of `artifacts`-themed inputs. Pure test-data refresh — the
   resolver itself is unchanged.
5. **React/TS** (12 files):
   - `ArtifactPolicyContext.tsx` shrinks substantially (170 lines
     deleted from the test file at `:225-396`); the whole "if path is
     under `artifacts/`, render this UX" branch is gone.
   - `useArtifactLinkHandler.ts:30-31` no longer imports/uses
     `useArtifactPolicyContext`.
   - `chatProjectContext.ts`, `sessionCwdSelection.ts`,
     `acpNotificationHandler.test.ts`, `acp.ts`, `acpApi.ts`,
     `chat.json` (i18n), `CreateProjectDialog.test.tsx`,
     `ToolCallAdapter.test.tsx`, `artifactPathPolicyCore.ts`,
     `artifactPathPolicy.test.ts` — all updated to drop references to
     `artifactsDir`/`artifactPathPolicy`/etc.

## Review

### Net direction is right

Special-casing a single sub-directory inside the user's project storage
created a leaky abstraction: the agent prompt couldn't actually steer
file operations because the UI had its own `artifacts/`-only opener
allowlist, and any file written outside that path needed a separate
`opener:allow-open-path` entry. Letting the prompt own where files go,
and relying on the broader `$HOME/.goose/**` (and other already-listed
roots) for the opener allowlist, is the cleaner shape.

### `default_serve_working_dir` change is the highest-risk piece

Pre-PR, `goose serve` with no `working_dir` override defaulted to
`$HOME/.goose/artifacts`. Post-PR it defaults to `$HOME`. That's a
behavior change for any caller that:
- Spawns the server without explicitly setting `working_dir`, AND
- Has the server invoke tools that write to relative paths.

Quick search of the diff doesn't show every spawn site, but
`goose_serve.rs:135` is reached when `default_serve_working_dir()` is
called as the fallback. **Recommend** the PR explicitly enumerate the
callers and confirm each passes an explicit working_dir, or change the
fallback to a path-not-yet-allocated-anywhere (e.g. a per-session
`$TMPDIR/goose-NNNN`) to avoid silently chrooting agent file writes
into `$HOME`. Putting agents loose in `$HOME` by default is a
significant blast-radius widening.

### `ProjectInfo.artifacts_dir` removal — back-compat

This is a public-ish struct serialized into `project.json` files via
serde derive (used by both `list_projects` deserialize at `:213-220`
and `create_project`/`update_project` write paths). Deleting the field
from `ProjectInfo` is fine for the read side (it just won't be present
in newly-emitted JSON) but the pre-existing `project.json` files on
disk will *still* contain `"artifacts_dir": "..."` from previous
installs. Confirm the `StoredProjectInfo` deserializer (which seems to
be a sibling type) either ignores unknown fields by default (serde
default for non-`deny_unknown_fields` types is to ignore) or has
`#[serde(default)]` on the dropped field. From the diff at `:165-168`
only `ProjectInfo` is touched — `StoredProjectInfo` is presumably
untouched and never had `artifacts_dir` (it was always derived). If
that's the case, no migration concern. Worth a one-line PR-body
confirmation.

### Capability narrowing is genuinely tighter

The `opener:allow-open-path` allowlist at
`capabilities/default.json:19-30` post-PR drops one specific glob and
relies on the broader `$HOME/.goose/**` to cover the same files. That's
strictly a *less* permissive change (no path that wasn't already
allowed becomes newly allowed) — the dedicated entry was redundant
once the broader glob was added. Good.

### Test deletions are appropriate

The 170 lines deleted from `ArtifactPolicyContext.test.tsx:225-396`
specifically tested the artifact-path-policy branching that no longer
exists. Keeping them would require keeping the code. The remaining
test file (~245 lines after deletion) still exercises the surviving
opener UX. Correct shape.

What's *not* in the diff: a positive test that confirms the new
"prompt-driven path handling" works end-to-end, i.e. that an agent
asking to write to `~/.goose/projects/X/foo.txt` still gets the file
created and openable. That's an integration-test surface and may live
in a separate suite; if so, fine.

### i18n cleanup

Confirm `chat.json` removal of artifact-related strings doesn't leave
dangling `t("...")` calls in code paths still active. The diff shows
the JSON file changes but a grep across the rest of the codebase for
the removed keys would catch any orphan references.

## Verdict

**merge-after-nits**

Wants:
1. **Confirm `default_serve_working_dir` blast-radius**: enumerate
   every spawn caller and verify each passes an explicit `working_dir`,
   or change the fallback to a per-session temp dir rather than
   `$HOME`. Defaulting agent file writes to the user's home dir is a
   meaningful permissions change worth calling out in the PR body.
2. **Confirm `StoredProjectInfo` ignores unknown fields**: existing
   on-disk `project.json` files written by earlier versions will still
   contain `"artifacts_dir"`. If serde rejects unknown fields, this
   PR breaks every existing install. One-line confirmation in the PR
   body.
3. **Grep for orphan `t("artifact...")` references** after the i18n
   key deletions in `chat.json`, to make sure no UI string still
   tries to look up a removed key.
4. **Optional**: mention in the PR body what prompt-engineering change
   accompanies this PR — "leaving that up to the prompting" implies a
   companion update to system prompts that previously assumed the
   `artifacts/` convention.

The cleanup direction is right and the file-by-file consistency is
good (`ProjectInfo`, capability allowlist, opener UX, i18n, all moved
together).
