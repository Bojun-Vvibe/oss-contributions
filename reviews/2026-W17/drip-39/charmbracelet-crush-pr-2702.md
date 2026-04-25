# charmbracelet/crush #2702 — feat: super yollo

- **Repo**: charmbracelet/crush
- **PR**: [#2702](https://github.com/charmbracelet/crush/pull/2702)
- **Head SHA**: `64a2da534c0ba3f877e67fcd8be329b35f84e108`
- **Author**: taciturnaxolotl (Kieran Klukas)
- **State**: OPEN (+378 / -37)
- **Verdict**: `needs-discussion`

## Context

Splits the existing `--yolo` (auto-approve everything) into a
three-mode permission model:
- `Normal`: prompt for all non-safe commands (today's default).
- `Yolo`: auto-approve non-dangerous commands, still prompt
  for dangerous ones (the new default for `-y`).
- `SuperYolo`: auto-approve everything including dangerous
  commands (the old `-y` semantics, now opt-in deeper).

"Dangerous" is defined by the existing `blockFuncs()` checks —
sudo, curl, npm `--global`, pip `--user`, brew install, go test
`-exec`, etc.

## Design

Cross-cutting touch across permissions, agent tools, shell, UI:

1. **`internal/permission/permission.go:18-31`** — new
   `PermissionMode` enum with three values. `Service`
   interface gains `SetPermissionMode` /
   `PermissionMode` getters (must update every implementor —
   diff shows mock updates in `bash_test.go:39-43` and
   `multiedit_test.go:37-41`, plus presumably the real impl).

2. **`internal/permission/permission.go:38-58`** — adds
   `Dangerous bool` to `CreatePermissionRequest` and
   `PermissionRequest`. Caller stamps it; permission service
   decides what to do with it based on mode.

3. **`internal/agent/tools/bash.go:191-194`** — new
   `isDangerousCommand` helper, just a thin wrapper around
   `shell.IsCommandBlocked(command, blockFuncs())`. Existing
   `blockFuncs()` is reused as the "dangerous" detector.

4. **`internal/agent/tools/bash.go:226-238`** — bash tool
   computes `isDangerous := isDangerousCommand(params.Command)`
   before requesting permission and stamps `Dangerous:
   isDangerous` on the request.

5. **`internal/agent/tools/bash.go:255` and `:311`** — the
   `bgManager.Start` calls now pass `nil` block functions
   instead of `blockFuncs()`. Comment at line 254: "No block
   functions - permission check handles dangerous commands."
   This is the **load-bearing change** of the whole PR: blocks
   used to *prevent* execution; now they only *classify*. The
   permission system has to be the actual gate.

6. **`internal/cmd/root.go:55`** — flag help text changes from
   "Automatically accept all permissions (dangerous mode)" to
   "Auto-approve non-dangerous commands, prompt for dangerous
   ones". The `-y` short-flag's *meaning* changes; users with
   muscle memory for "yolo = no prompts ever" will be
   surprised.

7. **`internal/agent/tools/bash.tpl:11`** — agent system prompt
   updated: "Banned commands return error" → "Dangerous
   commands require explicit user approval with a warning
   popup, even in YOLO mode." Important — model behaviour
   shifts because the model is told the rules changed.

8. **Tests**: `bash_test.go:46-128` adds
   `TestIsDangerousCommand` table-driven test with 12 cases
   covering the realistic boundaries (sudo, curl, npm `-g`,
   pip `--user`, brew install, go test `-exec`, etc., plus
   negatives like `ls`, `git status`).

## Risks

- **Behaviour change for existing `-y` users.** The flag's
  semantics flip: yesterday `crush -y` meant "ask me nothing";
  today it means "ask me about dangerous things". This is
  arguably the *right* default but it's a silent breaking
  change for anyone running `crush -y` in unattended pipelines
  expecting no prompts. They'll start hanging on dangerous
  commands.
- **No `--super-yolo` flag visible in the diff.** The enum has
  three modes but I only see `-y` mapped to `Yolo` (root.go:55)
  — how does a user opt into `SuperYolo`? Either the diff is
  incomplete (likely; we saw 16 files changed but only sampled
  the head of each) or the feature ships without a way to
  reach the most-permissive mode, which would be a regression
  for the existing `-y` behaviour.
- **`bgManager.Start(..., nil, ...)`** at lines 255, 311
  removes the *block* enforcement entirely. If the permission
  service has a bug (or the user picks `SuperYolo`),
  previously-blocked commands now run. The block was a
  belt-and-suspenders defence; this PR cuts the suspenders.
  Worth keeping `blockFuncs()` as a hard floor that even
  `SuperYolo` can't bypass for a small set of truly destructive
  things (`rm -rf /`, `dd of=/dev/sda`, etc).
- **PR title "super yollo" with double-l** is a typo in the
  branding — if the user-facing flag is `--super-yolo` the
  title should match. If it's `--super-yollo` then... yes.
- **Big surface (16 files, 378 additions) for a behaviour
  change** — should be split into:
  - "refactor: introduce PermissionMode enum"
  - "feat: classify commands as dangerous in
    permission requests"
  - "feat(cli): split yolo into yolo / super-yolo"
  Easier to bisect, easier to revert one piece.
- **No integration test for the "Yolo + dangerous → prompt"
  flow.** `TestIsDangerousCommand` proves classification but
  not the dispatch. Worth a test that wires
  `PermissionMode = Yolo` + a dangerous command and asserts
  `permissions.Request` is *called* (not auto-skipped).
- **CHANGELOG / migration guide** should explicitly call out
  the `-y` semantics change, in big bold letters.

## Suggestions

- Confirm the `--super-yolo` opt-in flag exists and is wired
  to `PermissionModeSuperYolo`.
- Keep `blockFuncs()` as a hard floor in `bgManager.Start` for
  a small absolute-block list (truly destructive ops),
  separate from the dangerous-classification list. Two-tier:
  "block always" + "dangerous, ask in non-super modes".
- Split into 3 commits as listed above before merge.
- Add CHANGELOG note: **breaking** for unattended `-y` users.
- Add an integration test for the Yolo+dangerous prompt path.

## What I learned

Permission models with two modes (`prompt` / `auto-approve`)
inevitably grow a third (`auto-approve safe, prompt dangerous`)
once users get burned. The transition hazard is the existing
flag: it's tempting to flip its meaning to the new sensible
default, but that breaks every shell script that depends on
the old semantics. The lower-pain path is to add a new flag
(`--auto-safe` or `--smart-yolo`) and leave `-y` meaning what
it always meant, until a major version. This PR takes the
"flip the flag" path — defensible, but should be loud about it.
