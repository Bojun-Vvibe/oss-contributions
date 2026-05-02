# charmbracelet/crush #2739 — fix(config): clear incompatible reasoning options when switching models

- **Repo:** charmbracelet/crush
- **PR:** #2739
- **Head SHA:** `f9b72be6420c831a35cba50fbbab1e5e2bfde4ca`
- **Verdict:** merge-after-nits

## Summary

Fixes #2713. When a user switches from a reasoning-capable model
(thinking enabled, `reasoning_effort` set) to a non-reasoning model
(e.g. local Ollama llama3), the persisted `Think` and
`ReasoningEffort` carry over via `jsons.Merge()` and reach the API,
producing errors like `model does not support thinking` or
`reasoning_effort is not supported for this model`.

Fix in `internal/config/load.go:configureSelectedModels`:

- Large branch (lines 615-624): wraps `large.ReasoningEffort = ...`
  and `large.Think = largeModelSelected.Think` in
  `if model.CanReason { ... }`.
- Small branch (lines 666-672): identical guard for the small model.
  Also moves the late `small.Think = smallModelSelected.Think` (was
  unconditionally set after the temperature/penalty block at the old
  line 686) inside the new `CanReason` guard.

## Specific notes

- **`load.go:615` and `:666`:** worth a one-line comment pointing at
  the underlying `jsons.Merge()` reason for the stale options — without
  it, the next maintainer might wonder why the `if` is needed at all
  ("the user just set this, why would I clear it?"). The PR description
  has the explanation; copy a sentence into the code.
- **Behaviour gap:** the fix prevents *applying* stale reasoning
  options at load time, but the persisted `SelectedModel.Think /
  ReasoningEffort` in the on-disk config are not cleared. The next
  switch back to a reasoning-capable model resurrects them. That's
  probably the desired behaviour ("preserve user intent across
  switches"), but worth confirming — alternatively, also clear them
  in the persisted struct on the model swap action, not at load.
- **Test (`load_test.go:1507-1551`):** uses a single Ollama provider
  with `CanReason: false` and asserts both `Think == false` and
  `ReasoningEffort == ""` post-`configureSelectedModels`. Targeted.
  Consider also asserting the same for the small-model branch in a
  second sub-test for symmetry — easy follow-up.

## Rationale

Real bug, minimal fix, regression test in place. The behaviour-gap
question is a design point worth discussing in review comments
rather than blocking on.
