# charmbracelet/crush #2768 — feat: launch hyper beta

- SHA: `515f795519ce8d6651ac462ccba7b909697267e5`
- State: MERGED, +1/-22 across 3 files

## Summary

Removes the `hyper.Enabled()` feature flag (which gated on `HYPER`/`HYPERCRUSH`/`HYPER_ENABLE`/`HYPER_ENABLED` env vars) and unconditionally enables the Hyper provider/login path. Net effect: launching Hyper out of beta — no more env-var opt-in required.

## Notes

- `internal/agent/hyper/provider.go:20-30` (deletion): the `sync.OnceValue`-wrapped `Enabled` function is removed wholesale, along with the `strconv` import. Clean removal.
- `internal/cmd/login.go:72-74`: `loginHyper` no longer returns `"hyper not enabled"`; just proceeds to device auth. Fine — gating is conceptually moved to "is the user logged in / does config have a token".
- `internal/config/provider.go:93-95` and `:168-169`: `UpdateHyper` and the parallel `Providers` fan-out drop the `Enabled()` guard. The `customProvidersOnly` short-circuit is preserved.
- Risk: any user previously relying on the env vars to *disable* Hyper now has no kill switch. If Hyper has any side effects on first launch (cache warmup, network call to `hyper.BaseURL()`), this is now unconditional. PR title says "beta", so presumably acceptable, but a release note callout is warranted.
- Diff is small and surgical; no test changes, which is reasonable since the change is the deletion of a gate.

## Verdict

`merge-as-is` — already merged. Follow-up worth tracking: ensure release notes mention there is no longer an env-var opt-out, in case anyone was relying on `HYPER=0` to skip the provider.