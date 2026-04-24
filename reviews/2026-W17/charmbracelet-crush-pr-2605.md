---
pr_number: 2605
repo: charmbracelet/crush
head_sha: e8e6bd2683ffc1257db9fc21673ce16db168e5ca
verdict: request-changes
date: 2026-04-24
---

# charmbracelet/crush#2605 — `additional_dirs` + `restrict_to_project` for tool access

**What changed.** Adds two new `Options` fields in `internal/config/config.go` (lines 245–246):
- `AdditionalDirs []string` — extra directories the agent can read/list without permission prompts.
- `RestrictToProject bool` — when true, deny outright (no prompt) any tool path outside `WorkingDir ∪ AdditionalDirs`.

Wires a new `tools.DirRestrictions` struct (`internal/agent/tools/dircheck.go`, 50 lines) into every file-touching tool: `bash`, `download`, `edit`, `multiedit`, `ls`, `view`, `write`. `coordinator.go` (lines 462–490) builds the restrictions, expanding `~` and env-vars via the resolver. `view.go` adds `isInAdditionalDir` (line 436) which is symlink-resolving via `EvalSymlinks`.

**Why it matters.** Two real needs in one PR: (a) "let me read this shared lib without 50 prompts" and (b) "deny outright if the agent tries to escape the project."

**Concerns.**
1. **`isInAdditionalDir` returns `false` if the file doesn't exist** (`EvalSymlinks` fails — see view_test.go line 511 explicitly asserts this). That breaks `Write` and `Edit` for **new** files inside `AdditionalDirs`: the path doesn't exist yet, so `EvalSymlinks` errors, so `isInAdditionalDir` is false, so in restricted mode the write is denied even though the user explicitly allowlisted that directory. This is a real bug for the documented use case ("`additional_dirs: [~/projects/shared-lib]` … then ask the agent to add a file"). Resolve symlinks on the *parent* directory, not the leaf path.
2. **`DirRestrictions` is passed by value to every tool** — the struct holds a `[]string` slice header. Cheap, but mutating `AdditionalDirs` after construction wouldn't propagate. Document immutability or pass `*DirRestrictions`.
3. **`bash.go` checks `execWorkingDir` only** (line 119) — but `bash` can `cd` mid-command or run `cat /etc/passwd` regardless of cwd. The restriction blocks the tool's working directory, not what the shell does. Either document this as a known gap or refuse to land "restrict_to_project" framing without a stronger guarantee — users will assume it sandboxes shell commands and it does not.
4. **`ls.go` line 365** has a tricky precedence: `if (err != nil || strings.HasPrefix(relPath, "..")) && !isInAdditionalDir(...)`. Reads OK, but a comment explaining "outside working dir AND not in additional dir" would help.
5. **`coordinator.go` resolver call** (line 460) silently swallows resolution errors (`if resolved, err := ...; err == nil`). If `$FOO` doesn't expand, the literal `$FOO` ends up in the allowlist — almost never what the user wants. Log a warning.
6. **No test for the new-file-in-additional-dir case** — directly tied to (1).

Conceptually the right feature, but `restrict_to_project` over-promises against `bash`, and the symlink-resolves-leaf bug breaks the headline use case. Fix both before merging.
