# ollama/ollama PR #15773 — cmd: show help when stop and show are called without a model argument

- **Repo:** ollama/ollama
- **PR:** [#15773](https://github.com/ollama/ollama/pull/15773)
- **Head SHA:** `20e933d12f222816b231be2cc8b0743cecacd79c`
- **Author:** anishesg (anish)
- **Size:** +27/-10 across 1 file
- **Reviewer:** Bojun (drip-24)

## Summary

Fixes #15769. `ollama stop` and `ollama show` both used
`cobra.ExactArgs(1)`, which returns the unhelpful error
`accepts 1 arg(s), received 0` when invoked without a model
name. Worse, the `PreRunE: checkServerHeartbeat` ran first, so
users saw a *server connection error* before they saw the args
error — making the failure look like an ollama server issue
rather than a missing argument.

The fix:
- Switch both commands to `cobra.MaximumNArgs(1)`.
- Wrap `PreRunE` to skip `checkServerHeartbeat` when `len(args)
  == 0`.
- Add an early return in `StopHandler` and `ShowHandler` that
  prints `cmd.Help()` when no args.

## Key changes

### `cmd/cmd.go` (+27/-10)

`StopHandler` at line 443:

```go
func StopHandler(cmd *cobra.Command, args []string) error {
+   if len(args) == 0 {
+       return cmd.Help()
+   }
    opts := &runOptions{
        Model:     args[0],
```

`ShowHandler` at line 1047:

```go
func ShowHandler(cmd *cobra.Command, args []string) error {
+   if len(args) == 0 {
+       return cmd.Help()
+   }
+
    client, err := api.ClientFromEnvironment()
```

The cobra command setup at line 2185:

```go
showCmd := &cobra.Command{
    Use:     "show MODEL",
    Short:   "Show information for a model",
-   Args:    cobra.ExactArgs(1),
-   PreRunE: checkServerHeartbeat,
-   RunE:    ShowHandler,
+   Args:    cobra.MaximumNArgs(1),
+   PreRunE: func(cmd *cobra.Command, args []string) error {
+       if len(args) == 0 {
+           return nil
+       }
+       return checkServerHeartbeat(cmd, args)
+   },
+   RunE:    ShowHandler,
}
```

Same pattern at line 2233 for `stopCmd`.

The two-layer guard (PreRunE skip + handler early return) is
correct — neither alone would suffice:

- Without the PreRunE skip, `checkServerHeartbeat` would still
  run and fail with a server-connection error before the handler
  printed help.
- Without the handler early return, calling
  `cobra.MaximumNArgs(1)` with zero args would happily proceed
  into the handler and `args[0]` would panic.

## Concerns

1. **`cobra.MaximumNArgs(1)` is too permissive.**

   It allows 0 or 1 args. But it also doesn't distinguish "1
   arg passed" from "user typo'd `ollama show foo bar`" — the
   latter falls back to `cobra`'s default behavior (probably
   silently dropping `bar`). A custom `Args` function:

   ```go
   Args: func(cmd *cobra.Command, args []string) error {
       if len(args) > 1 {
           return fmt.Errorf("accepts at most 1 arg, received %d", len(args))
       }
       return nil
   },
   ```

   would preserve the strict-mode behavior for the >1 case
   while allowing the 0-arg help case. Not blocking — current
   ollama UX probably accepts that "extra args are ignored".

2. **`cmd.Help()` returns an `error` from `RunE`.**

   `cobra`'s `cmd.Help()` returns `nil` on success and an error
   only if writing to stdout fails. So `return cmd.Help()` is
   correct — the help text gets printed and the process exits
   with code 0. Good — matches what `ollama --help`'s exit code
   would be.

3. **The PreRunE wrapper duplicates logic across two commands.**

   The same `if len(args) == 0 { return nil }; return
   checkServerHeartbeat(cmd, args)` pattern is repeated for
   both `showCmd` and `stopCmd`. A helper:

   ```go
   func skipHeartbeatIfNoArgs(cmd *cobra.Command, args []string) error {
       if len(args) == 0 {
           return nil
       }
       return checkServerHeartbeat(cmd, args)
   }
   ```

   would dedup this and make it harder to drift.

4. **No test coverage.**

   The PR description says "tested manually" but ships no test.
   A small test that builds the cobra command, invokes it with
   no args, and asserts:

   - exit code is 0,
   - stdout contains usage text,
   - `checkServerHeartbeat` was not called (would need the
     heartbeat call to be injectable),

   would protect against a future refactor (e.g., someone
   re-tightening `Args` back to `ExactArgs(1)` because they
   didn't notice the help case). For a UX-focused fix this
   would be fairly easy to write.

5. **What about `ollama run` and `ollama pull`?**

   The PR fixes `stop` and `show` but `ollama run`, `ollama
   pull`, `ollama push`, `ollama delete`, and `ollama cp` likely
   have the same `ExactArgs` + `PreRunE: checkServerHeartbeat`
   pattern. If the goal is "no model name → show help, not
   server error", that should be applied uniformly. A
   follow-up PR with a sweep would be valuable.

6. **Minor formatting drift.**

   The new code has `Use:  "show MODEL",` (two spaces) where the
   surrounding cobra commands use `Use:     "show MODEL",`
   (multiple spaces aligned). `gofmt` will normalize this on
   the next save, but worth running `gofmt -w` before merge.

## Verdict

`merge-after-nits` — solid UX fix that closes a real issue.
The two-layer guard is correct, the `cmd.Help()` return value
handling is right, and the PreRunE wrapper is the minimum-
intrusive way to skip the heartbeat check. Five small
follow-ups:

- factor the PreRunE wrapper into a `skipHeartbeatIfNoArgs`
  helper used by both commands,
- add a unit test that asserts no-args produces help text + zero
  exit (without a running server),
- file a follow-up issue to apply the same pattern to `run`,
  `pull`, `push`, `delete`, `cp`,
- run `gofmt -w` on the new struct literals,
- consider a custom `Args` function that rejects >1 args
  explicitly.

## What I learned

The "PreRun fails before the user sees the actual error" class
of CLI bug is sneaky because it makes failures *look* like a
different problem (server unreachable instead of missing
argument). The fix pattern — skip the network probe when the
call is going to fail validation anyway — generalizes: any CLI
that does eager preflight checks before argument validation
risks making the user blame the wrong subsystem. The cleaner
architecture is "validate args first, then check preconditions
that depend on those args". cobra's `PreRunE` running before
`Args` validation is a footgun that this PR works around at
the cost of two layers of guards instead of one.
