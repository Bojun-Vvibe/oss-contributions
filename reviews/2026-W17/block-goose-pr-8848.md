# block/goose #8848 — feat(goose2): migrate git commands from Tauri to ACP+ (#8698)

- **Repo**: block/goose
- **PR**: #8848
- **Author**: fresh3nough
- **Head SHA**: 261a9fca06292c3f8f879d6b20e5ca59ecb508b2
- **Base**: main
- **Size**: +1296 / −282 — primarily 9 new ACP RPCs and matching
  request/response DTOs in
  `crates/goose-sdk/src/custom_requests.rs` (+~250), schema
  regeneration (`acp-meta.json` +45, `acp-schema.json` +315), plus
  the desktop-side migration of `goose2/`'s git commands.

## What it changes

Adds 9 RPCs under the `_goose/git/*` namespace, replacing the
Tauri-side git commands the desktop UI used to call directly:

- `_goose/git/state` → `GitStateResponse { state: GitState }` with
  `currentBranch`, `dirtyFileCount`, `incomingCommitCount`,
  `worktrees`, `isWorktree`, `mainWorktreePath`, `localBranches`
  (`custom_requests.rs:483-505`).
- `_goose/git/changed_files` → `Vec<ChangedFile>` with `status`
  ∈ `added | modified | deleted | renamed | copied | untracked`,
  plus `additions`/`deletions` counts (`:475-481`).
- Five mutating RPCs returning `EmptyResponse`:
  `switch_branch`, `stash`, `init`, `fetch` (with `--prune`),
  `pull` (`--ff-only`), `create_branch`
  (`:541-585`).
- `_goose/git/create_worktree` (`:597-612`) takes
  `{ path, name, branch, createBranch (default false), baseBranch (optional) }`
  and returns `CreatedWorktree { path, branch }`. Docstring clarifies:
  `createBranch=true` requires `baseBranch`; `createBranch=false`
  requires `branch` to already exist.

## Strengths

- The Rust DTOs treat each git operation as a typed request with a
  named response type — no stringly-typed JSON blobs across the
  ACP boundary. The schema regen lines up cleanly: every RPC's
  request and response shape is in `acp-schema.json` with
  `x-side: agent` + `x-method` cross-references at
  `acp-schema.json:1067-1380`.
- `GitState` is a *snapshot*, not a stream of events
  (`custom_requests.rs:483`). Right call — the renderer doesn't
  need real-time updates of the dirty file count, and a snapshot
  RPC is cache-friendly and idempotent. A future "watch" RPC can
  be added without breaking this one.
- `pull` is hardcoded to `--ff-only` in the docstring at `:573` —
  the right default for a UI button. No silent merges.
- `fetch --prune` (`:565`) similarly does the right thing for a UI
  refresh: removes stale tracking branches the user might still be
  hovering in the local-branches list.
- `create_worktree` separates `branch` (always required) from
  `createBranch` (boolean) and `baseBranch` (only meaningful when
  creating). The docstring at `:589-594` spells out the contract,
  which is the tricky part.

## Concerns / asks

- All five mutating ops return `EmptyResponse`. For `switch_branch`
  and `stash`, the caller almost always wants the resulting state
  (new HEAD, stash ref). Returning `EmptyResponse` forces a
  follow-up `_goose/git/state` round-trip on every single mutation.
  Two options:
  1. Return the new `GitState` from each mutating RPC (small extra
     work server-side, much cheaper for the UI).
  2. At minimum, document that callers are expected to call `state`
     after every mutation, and consider an `_goose/git/atomic_apply`
     that bundles a mutation + state read.
- `status` field on `ChangedFile` (`:478`) is a free-form string. The
  docstring lists 6 valid values but the schema doesn't enum-constrain
  them — same issue as the sibling PR #8849's `kind` field. Worth
  generating an `enum: [...]` JSON-schema constraint so a
  TypeScript SDK consumer gets a discriminated union, not `string`.
- `incomingCommitCount` (`:495`) is `u32`. What if the local branch
  has *outgoing* commits (ahead of remote)? The DTO names suggest
  fetching, but the field doesn't disambiguate "ahead vs behind". A
  separate `outgoingCommitCount` (or an `ahead`/`behind` pair) would
  surface the more useful "your branch is ahead by N" signal that
  the UI almost certainly wants for a status bar.
- `create_worktree`'s `createBranch=false` + `branch=existing` path:
  what if `branch` exists but is already checked out somewhere else
  (git's hard error: "fatal: branch is already checked out")? The
  docstring doesn't promise behavior. Worth either:
  - documenting that this RPC bubbles up the git error verbatim
    (`fatal: ...`), or
  - normalizing the error into a structured `WorktreeError {
    kind: "BranchAlreadyCheckedOut", existingPath }` variant the UI
    can surface meaningfully.
- `GitInitRequest` (`:557`): does it bail if `path` is already a
  git repo, or does it silently succeed? `git init` itself is a
  no-op on an existing repo (just reports "Reinitialized"), but
  the UI probably wants the distinction. Worth a `wasNew: bool` in
  a richer response or a doc note.
- The `worktrees` list in `GitState` (`:486`) is unbounded. A repo
  with hundreds of worktrees (CI farms, sub-agent worktrees) could
  inflate every `state` call. Worth a `worktreeLimit` or just
  documenting that this RPC isn't intended for high-cardinality
  cases.

## Verdict

**merge-after-nits** — the namespace grouping, snapshot-vs-stream
choice, and `create_worktree` parameter design are all solid. The
asks cluster around two themes:

1. Mutating RPCs that drop their result on the floor — folding the
   post-state read back into the response saves a round-trip and
   eliminates a TOCTOU window between mutate and refresh.
2. Stringly-typed enums (`status` on `ChangedFile`) and
   under-specified error shapes (worktree-already-checked-out, init
   on existing repo) — closing them now while these RPCs are brand
   new is much cheaper than later.

## What I learned

When migrating from in-process commands (Tauri `invoke`) to
out-of-process RPC, the *number of round trips* becomes a real cost.
Tauri commands could afford to be tightly scoped because the
overhead of `mutate; then read state` was negligible — over JSON-RPC
the same pair of calls is two serialization passes and two
deserialization passes per UI action. That's why the convention
`mutate(...) -> NewState` tends to win in network-protocol designs
even when "single-responsibility" instinct says the mutator
shouldn't read.
