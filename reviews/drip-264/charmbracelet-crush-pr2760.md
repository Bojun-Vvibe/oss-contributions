# charmbracelet/crush PR #2760 — feat(bedrock): add aws_auth_refresh option

- **URL**: https://github.com/anomalyco/crush/pull/2760
- **Head SHA**: `1bd7ba6dbe7f`
- **Files touched**: `internal/agent/coordinator.go`, `internal/agent/bedrock_auth_test.go` (new), `internal/config/config.go`, `internal/config/load.go`, `internal/config/aws_auth_refresh_test.go` (new)

## Summary

Adds a string `Options.AWSAuthRefresh` config (`config.go:260`) that names a shell command. The command runs (a) at startup before configuring the Bedrock provider (`load.go:289`), and (b) once on retry if a Bedrock call returns 401/403, classified by new helper `coordinator.isBedrockAuthError` (`coordinator.go:965-975`). `RunAWSAuthRefresh` (`load.go:755-765`) wires stdin/stdout/stderr to the terminal so SSO/MFA prompts work.

## Comments

- `load.go:755-765` — `exec.Command("sh", "-c", cmd)` with `cmd` taken verbatim from user config is fine here (config is local, user-trusted), but worth a code comment stating that explicitly so a future contributor doesn't paste this pattern into a context where config comes from over the wire.
- `coordinator.go:236-247` — the retry path runs `RunAWSAuthRefresh` synchronously then re-invokes `run()`. There's no backoff and no second-failure guard: if the refresh succeeded but credentials are still bad (e.g. SSO returned an unrelated profile), we retry exactly once and return `originalErr`. That's fine, but log the post-refresh error too so users can distinguish "refresh failed" from "refresh succeeded but Bedrock still 401".
- `coordinator.go:973-975` — `isBedrockAuthError` returns true for *any* 403 from Bedrock. Bedrock returns 403 for `AccessDeniedException` (genuinely unauthorized model) too, not just `ExpiredTokenException`. Re-running `aws_auth_refresh` for an `AccessDeniedException` will burn an interactive SSO prompt that can't help. Inspect the response body for the exception code if available.
- `bedrock_auth_test.go:18-37` — solid table test, but every `ProviderError` is constructed without the actual error body. Add a case where the body says `AccessDeniedException` (once the code differentiates) to lock in the narrower behaviour.
- `aws_auth_refresh_test.go:23` — `RunAWSAuthRefresh("test 1 = 1 && exit 0")` confirms shell expansion. Also add a test for a command that *prints* to stderr to confirm output is forwarded (the docstring promises this).
- `config.go:260` — JSONSchema example `exec duo-sso -profile myprofile -valid-session-threshold 7200` is great. Consider adding a second, simpler example like `aws sso login --profile mine` for the common case.

## Verdict

`merge-after-nits` — clean feature, well-tested. The 403-overmatch on `AccessDeniedException` is the only real correctness wart; the rest is polish.
