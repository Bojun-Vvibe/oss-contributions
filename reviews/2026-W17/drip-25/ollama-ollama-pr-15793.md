# ollama/ollama PR #15793 — mlx: update to 0.31.2

- **Repo:** ollama/ollama
- **PR:** [#15793](https://github.com/ollama/ollama/pull/15793)
- **Head SHA:** `bdd7fdd171be290099d368aacb747f9a1241299a`
- **Author:** pd95 (Philipp)
- **Size:** +1341 / −126 across 19 files
- **Reviewer:** Bojun (drip-25)

## Summary

Vendored MLX/MLX-C dependency bump to MLX 0.31.2, plus refreshed
generated C bindings for the MLX runner. The motivating fix is
`ml-explore/mlx#3422` — the NAX `addmm` broadcast-bias regression
where `bias.Addmm(x, w, 1, 1)` silently dropped the broadcast bias
term for batched inputs. Ollama exposes and uses MLX `addmm` for
biased linear layers in three places:

- `x/models/nn/nn.go` — `Linear.Forward` uses
  `l.Bias.Addmm(x, w, 1.0, 1.0)`
- `x/mlxrunner/mlx/nn.go` — same
- `x/imagegen/mlx/mlx.go` — exposes `AddMM` to imagegen

So this is a "real fix at the dependency layer that preserves
existing call sites" upgrade rather than a routine refresh.

The PR commits are intentionally ordered for reproducibility:
1. `mlx: test addmm broadcast bias` (commit `75f395c`) — adds the
   regression test that fails on pre-0.31.2.
2. `mlx: update to 0.31.2` (commit `bdd7fdd`) — bumps `MLX_VERSION`
   and `MLX_C_VERSION`, regenerates bindings, makes the test pass.

## Key changes

### Pin updates

- `MLX_VERSION`: `38ad257088fb2193ad47e527cf6534a689f30943` →
  `68cf2fddd8de5edd8ab3d926391772b2e2cedad8`
- `MLX_C_VERSION`: `0726ca922fc902c4c61ef9c27d94132be418e945` →
  `fba4470b89073180056c9ea46c443051375f7399`

### Regenerated bindings

- `x/mlxrunner/mlx/generated.{c,h}` (+158 / +391, −5 / −25)
- `x/mlxrunner/mlx/include/mlx/c/*` — new headers for
  `compile.h`, `distributed_group.h`, `fft.h`, `graph_utils.h`,
  `io.h`, `io_types.h`, `mlx.h`, `ops.h`
- `x/imagegen/mlx/mlx.{c,h}` (+339 / +146, −49 / −30)

### Thread-local stream model: `x/mlxrunner/mlx/stream.go` (+4 / −2), `dynamic.go` (+11 / 0), `mlx.go` (+1 / −4), `runner.go` (+1)

The substantive runtime change. MLX 0.31 makes default streams
and Metal command encoders thread-local. Since Go goroutines can
move between OS threads, a lazy MLX graph could be built on one
thread's stream and evaluated on another, producing:

> `mlx: There is no Stream(gpu, N) in current thread`

The wrapper now pins MLX work to its OS thread (via
`runtime.LockOSThread()`) and resolves the default stream
per-thread instead of caching it globally.

The PR body has the load-bearing caveat:

> The OS-thread pin is intentionally not paired with
> `runtime.UnlockOSThread()`: MLX lazy graph construction and
> later evaluation need to stay on the same OS thread for the
> lifetime of that MLX goroutine.

That's the right call given MLX's threading model, but it
deserves a multi-line code comment at the `LockOSThread()` site
explaining why the unlock is omitted — without that, a future
maintainer doing a "every Lock should have an Unlock" audit will
add the unlock back and silently break long-running MLX
sessions.

### Regression test: `x/mlxrunner/mlx/ops_test.go` (+87 / 0)

New `TestAddmmBroadcastBiasMatchesMatmulAdd` asserts:

```
go test ./x/mlxrunner/mlx -run TestAddmmBroadcastBiasMatchesMatmulAdd -count=1 -v
```

Uses zero input to isolate the broadcast-bias term. On the old
MLX, `bias.Addmm(x, w, 1, 1)` matches matmul-only (proving the
bias is dropped); on 0.31.2, addmm matches matmul + add.

The reviewer reproduction flow in the PR body is unusually
clean — checkout the test commit, build, see the bug; checkout
the upgrade commit, build, see the fix.

## Concerns

1. **`runtime.LockOSThread()` without `UnlockOSThread()` is a
   maintenance footgun.**
   The decision is correct for MLX's thread-local stream model,
   but the missing unlock will trip every future Go-runtime
   audit. Strongly recommend a multi-line comment block at the
   call site:
   ```go
   // Pin this goroutine to its OS thread for the lifetime of
   // the MLX session: MLX 0.31+ uses thread-local default
   // streams and Metal command encoders, and a lazy graph built
   // on one thread cannot be evaluated on another (will produce
   // "There is no Stream(gpu, N) in current thread"). We
   // intentionally do NOT call runtime.UnlockOSThread() — the
   // pin must persist for the entire MLX goroutine's lifetime.
   runtime.LockOSThread()
   ```
   Without this, the next contributor doing a goroutine-leak
   audit will add the unlock back.

2. **Generated-binding diff is large and not human-reviewable.**
   1300+ lines across `generated.c`/`generated.h` plus 8 new
   header files is mechanical and shouldn't get a line-by-line
   review. But: there's no checked-in script that regenerates
   these bindings, just a `./scripts/build_darwin.sh -a arm64
   build` reference. Worth landing a `make regen-mlx-bindings`
   target (or equivalent) so a future maintainer doing the next
   MLX bump can produce the same diff deterministically.

3. **First-time contributor + AI-assisted investigation.**
   The PR body discloses upfront:
   > Note: This is my first contribution to Ollama. I used
   > Codex to help investigate the MLX `addmm` issue, create
   > the focused regression test, and prepare this PR. I
   > manually reproduced the fail/pass behavior on macOS
   > before marking this ready for review.

   The disclosure is the right shape (declared, evidence
   provided, manual repro confirmed). The contribution itself
   is well-structured (commit ordering, regression test that
   *demonstrates the bug before the fix*, clean reviewer repro
   flow). No concern about the contribution shape — flagging
   only because the reviewer should verify the test actually
   exercises the bug (which it does — zero input + non-zero
   bias is the right minimum-witness construction).

4. **MLX 0.31 thread-local stream change has wider blast radius
   than addmm.**
   Any other code path that constructs an MLX graph on one
   goroutine and evaluates on another will break. The PR pins
   the MLX runner goroutines, but if Ollama spawns helper
   goroutines that touch MLX (e.g. for telemetry, progress
   reporting, cancellation), those will now produce the same
   "no Stream in current thread" error. Worth a grep for
   other MLX call sites outside the wrapper to confirm none
   of them are reachable from non-pinned goroutines.

5. **No CI verification on Linux for the addmm test.**
   PR body notes: "In the Linux container, the same targeted
   go test command passes by skipping MLX runtime because the
   macOS MLX dylib is unavailable." That's expected, but means
   the regression test only actually runs on macOS arm64 CI.
   If macOS CI is flaky or expensive, the regression coverage
   for this bug class is thin. Worth confirming the macOS arm64
   matrix slot is non-optional and gates merges.

## Verdict

`merge-after-nits` — the dependency bump fixes a real
correctness bug (silent bias drop in batched linear forward
passes), the regression test demonstrates the bug before the
fix (textbook "test fails on broken commit, passes on fixed
commit" reviewer-repro flow), and the threading model
adjustment is correctly handled. Two pre-merge asks: a
multi-line comment at the `runtime.LockOSThread()` site
explaining why the unlock is omitted (prevents future
"missing unlock" regressions), and a checked-in `make
regen-mlx-bindings` target so the next bump is deterministic.
One follow-up worth filing: grep for other MLX call sites
outside `mlxrunner/` to confirm none are reachable from
non-pinned goroutines.

## What I learned

Dependency bumps that include silent-correctness fixes (like
"AddMM was dropping the broadcast bias term, here's a fix in
the new version") are dangerous because the bug is invisible
in any test that doesn't construct the minimum witness — most
real models *would* have nonzero input AND bias, so the bug
hides in the noise. The pattern this PR uses — landing the
regression test as a separate commit *before* the dep bump,
so reviewers can `git checkout` the test commit and see the
bug — is exactly the right shape. Same lesson applies to
floating-point reductions, accumulator order changes, and
any "the new version is more correct" upstream fix: build
the witness test as a separate commit, prove the bug is real
on the old version, then ship the fix. The other lesson is
the `LockOSThread()` without `UnlockOSThread()` pattern: when
the underlying library has thread-local state with goroutine-
lifetime semantics, the pin has to be a one-way operation,
and that contract has to be loud in the code or the next
maintainer will "fix" the missing unlock.
