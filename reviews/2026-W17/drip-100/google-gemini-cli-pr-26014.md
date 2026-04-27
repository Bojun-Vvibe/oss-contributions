# google-gemini/gemini-cli#26014 — fix(cli): randomize sandbox container names

- **Author**: Kkartik14
- **Size**: +92 / -25 (sandbox.ts + sandbox.test.ts)
- **Issue**: #12083

## Summary

The previous container-naming scheme in `packages/cli/src/utils/sandbox.ts:490-516` ran `docker ps -a --format "{{.Names}}"` to list existing containers, then picked the first unused sequential index `${imageName}-${N}`. This races when two CLIs start concurrently — both list, both see no `${name}-0`, both try to create `${name}-0`, second one fails.

Fix replaces the probe-and-increment with a 12-hex-char random suffix from `crypto.randomBytes(6)`, applied uniformly to both the regular branch (`${imageName}-<hex>`) and the integration-test branch (`gemini-cli-integration-test-<hex>`). The `docker ps -a` exec call is dropped entirely.

## Diff inspection

```diff
-    let containerName;
-    if (isIntegrationTest) {
-      containerName = `gemini-cli-integration-test-${randomBytes(4).toString('hex')}`;
-    } else {
-      let index = 0;
-      const containerNameCheck = (
-        await execAsync(`${command} ps -a --format "{{.Names}}"`)
-      ).stdout.trim();
-      while (containerNameCheck.includes(`${imageName}-${index}`)) {
-        index++;
-      }
-      containerName = `${imageName}-${index}`;
-    }
+    const containerNamePrefix = isIntegrationTest
+      ? 'gemini-cli-integration-test'
+      : imageName;
+    const containerName = `${containerNamePrefix}-${randomBytes(6).toString('hex')}`;
```

Suffix length went from 4 hex chars (16-bit) for integration-test → 6 hex chars (24-bit) uniformly. Birthday-collision threshold goes from ~256 concurrent containers (4-hex) to ~4096 (6-hex), which is comfortably above any realistic concurrent-CLI count.

Test coverage at `sandbox.test.ts:339-410` is thorough:
- `vi.mock('node:crypto', () => ({ randomBytes: vi.fn().mockReturnValue(Buffer.from('a1b2c3d4e5f6', 'hex')) }))` — deterministic suffix for assertions.
- New `mockedExecCommands: string[]` array captures every `exec` call and the test asserts `not.toEqual(expect.arrayContaining([expect.stringContaining('ps -a --format')]))` — directly verifies the probe was removed.
- `expect(spawn).toHaveBeenNthCalledWith(2, 'docker', expect.arrayContaining(['--name', containerName, '--hostname', containerName, '--env', `SANDBOX=${containerName}`]))` — confirms all three places that need the same name stay aligned.
- Symmetric integration-test variant at `:344-410` exercises the `gemini-cli-integration-test-` prefix branch.

## Strengths

- Eliminates a real race condition cleanly, no fallback hybrid (which would have re-introduced the probe).
- Removing the `docker ps -a` exec also removes a 50-200ms latency per sandbox start.
- 24-bit suffix is appropriate for the workload — birthday paradox bound at ~4k concurrent containers vs realistic worst-case ~10.
- Test uses deterministic `randomBytes` mock so assertions are stable across runs.
- Both `--name`, `--hostname`, and `SANDBOX` env var stay aligned (prevents the subtle case where container name changes but in-sandbox `$SANDBOX` lies).
- Integration-test prefix preserved — keeps CI cleanup scripts that grep `gemini-cli-integration-test-*` working.

## Concerns

1. **No collision-detection fallback.** With 24 bits of entropy, collisions are vanishingly unlikely but not impossible. If `docker run --name <existing>` fails, the user gets the raw docker error rather than a "retry with new suffix" path. Acceptable for now, but worth a comment noting "we accept the ~2^-24 retry-by-rerunning UX" so the next reviewer doesn't try to "fix" it by reintroducing the probe.
2. **`randomBytes(6)` is sync.** Fine here (called once per sandbox start, microsecond cost), but if the sandbox-start path ever moves into an async-batched context, switching to `randomBytes(6, cb)` would be cheap insurance. Not a blocker.
3. **Old container leak.** The previous "find first unused index" implicitly capped name-space growth: `name-0` could be reused after `docker rm`. The new random-suffix approach means old `docker ps -a` listings will accumulate dead `name-<hex>` entries forever (unless `--rm` is in the docker invocation, which the existing `expect.arrayContaining(['run', '-i', '--rm', '--init'])` confirms it is). Worth a one-line comment that `--rm` is what makes name-leak non-issue.
4. **PR doesn't mention podman compatibility.** `randomBytes(6).toString('hex')` works for both, but if `command` can be `podman` and podman has its own naming-convention restrictions (it doesn't — same rules as docker), a one-line note in the PR body would settle it.

## Verdict

**merge-as-is** — the race fix is correct, the suffix-length math is sound, and the test pins both the dropped probe and the symmetric prefix handling. The collision-fallback comment and `--rm`-justifies-leak comment are nits that can ride on a follow-up commit.
