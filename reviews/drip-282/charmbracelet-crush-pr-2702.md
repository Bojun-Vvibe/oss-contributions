# charmbracelet/crush PR #2702 — feat: super yolo

- Head SHA: `64a2da534c0ba3f877e67fcd8be329b35f84e108`
- Size: +378 / -37, 16 files

## Specific refs

- `internal/agent/tools/bash.go:191-194` — new `isDangerousCommand()` reuses existing `shell.IsCommandBlocked(command, blockFuncs())`; this becomes the single source of truth for "needs prompt even in YOLO".
- `internal/agent/tools/bash.go:226-238` — in YOLO/non-safe-readonly path, command goes through `permissions.Request` with new `Dangerous: isDangerous` flag instead of being silently auto-approved.
- `internal/agent/tools/bash.go:255,311` — `bgManager.Start(...)` now passed `nil` for blockFuncs because permission layer handles it. **Concern:** this means if permissions layer is bypassed (super-yolo path), `blockFuncs` no longer enforces anything — there is no defense in depth.
- `internal/permission/permission.go:+52/-1` — adds `PermissionMode` (Normal/Yolo/SuperYolo), `SetPermissionMode`, `PermissionMode()`, and `Dangerous` field on requests.
- `internal/agent/tools/bash_test.go:46-127` — new `TestIsDangerousCommand` table-driven test covering curl, sudo, npm -g, pip --user, brew, go test -exec.
- `internal/ui/dialog/permissions.go:+16` — warning banner styling for dangerous prompts.
- `internal/cmd/root.go:55` — wires the new flag/persistent option for super-yolo mode.

## Assessment

Direction is right: silent auto-approval of `curl | sh` and `sudo` in YOLO was a foot-gun that closes #2463 and #1451. The two-tier model (yolo prompts dangerous, super-yolo doesn't) is consistent with how other agents handle this.

Real concern: removing `blockFuncs()` from the bgManager.Start path (`bash.go:255,311`) gives up the in-shell block layer. Currently the only thing protecting super-yolo from a `rm -rf /` is the user themselves. Worth either (a) keeping blockFuncs for super-yolo too with an explicit allowlist for the dangerous set, or (b) documenting that super-yolo is intentionally unsandboxed.

Also: `permission_test.go:+76/-14` adds tests for the new mode plumbing but doesn't appear to test the super-yolo bypass path end-to-end. A test that confirms super-yolo skips the prompt for `curl https://...` would lock in the contract.

Test mocks updated in two places (`bash_test.go:39-44`, `multiedit_test.go:37-42`); the duplication smells but is contained.

verdict: needs-discussion