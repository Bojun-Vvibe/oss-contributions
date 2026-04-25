# ollama/ollama #15713 — Use MemAvailable equivalent in cgroup memory check

- **Repo**: ollama/ollama
- **PR**: [#15713](https://github.com/ollama/ollama/pull/15713)
- **Head SHA**: `f266fbf17c2edf620c1e760bebaf80c9633ce05f` (latest on branch)
- **Author**: see `gh pr view`
- **State**: OPEN
- **Verdict**: `merge-after-nits`

## Context

When running inside a container with a cgroup memory limit, ollama's
discovery layer reports `FreeMemory = TotalMemory - memory.current`. That
formula treats the entire `memory.current` value as "used and
unreclaimable", but on Linux a large fraction is typically inactive page
cache that the kernel will evict on demand. The result: ollama refuses
to load a model that would in fact fit, because it underestimates
available memory under cgroup v2. This PR replicates the
`/proc/meminfo:MemAvailable` accounting trick (subtract reclaimable file
cache from "used") for the cgroup path.

## Design

The change in `discover/cpu_linux.go:73-99` is small and correct:

```go
inactiveFile, _ := getCgroupMemStat("inactive_file")
reclaimable := min(inactiveFile, used)
mem.FreeMemory = mem.TotalMemory - used + reclaimable
```

Three things it gets right:

1. **Best-effort, never fail-loud.** The error from `getCgroupMemStat`
   is swallowed (`_`), and `inactiveFile` defaults to zero, so a missing
   `memory.stat` file or a malformed entry degrades to the previous
   conservative estimate — no regression for environments that don't
   expose the field. Same fail-open posture the surrounding code uses
   for `memory.current` itself at line ~74.

2. **`min(inactiveFile, used)` clamp** prevents the (theoretical) case
   where stale stat readings would make `reclaimable > used` and produce
   a `FreeMemory` larger than `TotalMemory`. Cheap defensive bound.

3. **Helper split.** `cgroupMemStatFromFile(path, key)` at lines 86-99
   takes a path, which is exactly what the new test
   `TestGetCgroupMemStatInactiveFile` in `cpu_linux_test.go:2086-2138`
   needs to exercise the parser against synthetic content without
   touching `/sys/fs/cgroup`. Three table rows: happy path with the
   real-world layout (`anon` / `file` / `inactive_file` / `active_file`),
   key absent, empty file. Good coverage of the parser; the wrapping
   `getCgroupMemStat` (which hardcodes the path) is the trivial
   adapter and reasonably untested.

## Risks / Nits

1. **`active_file` is also reclaimable under pressure.** The kernel's
   `MemAvailable` formula is not just `inactive_file` — it also adds a
   fraction of `slab_reclaimable` and capped `active_file` (see
   `mm/page_alloc.c:si_mem_available`). For a tighter approximation
   under heavy memory pressure, a follow-up could add `slab_reclaimable`
   too. Not blocking — `inactive_file` alone closes the bulk of the
   reported gap.

2. **Cgroup v1 not addressed.** The path
   `/sys/fs/cgroup/memory.stat` is the v2 layout. On v1 systems this
   file lives under `/sys/fs/cgroup/memory/memory.stat` with a
   different key spelling (`total_inactive_file`). The PR description
   should clarify this is v2-only; users on Kubernetes nodes with
   cgroup v1 still hit the original underestimate. If multi-version
   support is desired, a sniff at startup (look at
   `/sys/fs/cgroup/cgroup.controllers` to confirm v2) would be the
   cleanest gate.

3. **Test calls `f.WriteString(tt.content)` without checking the error**
   at `cpu_linux_test.go:2125`. Linters in this repo may complain;
   wrapping in `if _, err := f.WriteString(...); err != nil { t.Fatal(err) }`
   is the usual style.

## What I learned

The "available memory" question is one of those where the obvious
formula (`total - used`) is wrong on Linux for historical reasons —
`MemAvailable` exists in `/proc/meminfo` precisely because every monitor
that didn't account for page cache reclaim was lying to operators for a
decade. The fact that ollama got bitten by this in the cgroup path is a
nice reminder that platform-specific resource accounting needs to track
the platform's own definitions, not the lowest-common-denominator math.
