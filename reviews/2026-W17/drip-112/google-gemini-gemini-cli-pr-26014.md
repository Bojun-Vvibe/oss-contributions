# google-gemini/gemini-cli PR #26014 — fix(cli): randomize sandbox container names

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26014
- **Author**: @Kkartik14
- **Head SHA**: `343fc1ec758f887e5444d3191134f2f28f8c1489`
- **Size**: ~+90 / −20
- **Files**: `packages/cli/src/utils/sandbox.ts`, `packages/cli/src/utils/sandbox.test.ts`

## Summary

Race condition fix. The previous container-naming logic ran `docker ps -a --format '{{.Names}}'`, then linearly probed `${imageName}-0`, `${imageName}-1`, ... until it found an unused suffix. Two CLI invocations starting concurrently could both see the same probe result and pick the same `${imageName}-N`, producing a "name in use" error from Docker. The fix replaces the probe with a 6-byte random hex suffix (`${imageName}-${randomBytes(6).toString('hex')}`) for both regular and integration-test paths.

## Verdict: `merge-after-nits`

Correct fix for a real concurrency bug, no longer requires a `docker ps -a` round-trip per start (a small latency win), and the test rewrite is thorough — both the regular path and the `GEMINI_CLI_INTEGRATION_TEST=true` path have new assertions that include the `--name` / `--hostname` / `SANDBOX=...` env-var triple. One nit: the probe-removal also removes a small bit of "stable, predictable name" UX that some users may have relied on for `docker logs` / `docker exec` workflows.

## Specific references

- `packages/cli/src/utils/sandbox.ts:490-498` — the entire replacement:
  ```ts
  // Use a random suffix instead of probing existing containers so concurrent
  // CLI starts cannot race on the same sequential name.
  const imageName = parseImageName(image);
  const isIntegrationTest = process.env['GEMINI_CLI_INTEGRATION_TEST'] === 'true';
  const containerNamePrefix = isIntegrationTest
    ? 'gemini-cli-integration-test'
    : imageName;
  const containerName = `${containerNamePrefix}-${randomBytes(6).toString('hex')}`;
  ```
  6 bytes hex = 12 hex chars = 48 bits of entropy. Collision probability between two concurrent starts is negligible (~1 in 2^48). Note the bump from the previous integration-test code's `randomBytes(4)` (32 bits) to `randomBytes(6)` (48 bits) — the test fixture at line 109 even pins this: `expect(randomBytes).toHaveBeenCalledWith(6)`.
- The deleted block at the same location previously did:
  ```ts
  const containerNameCheck = (await execAsync(`${command} ps -a --format "{{.Names}}"`)).stdout.trim();
  while (containerNameCheck.includes(`${imageName}-${index}`)) { index++; }
  containerName = `${imageName}-${index}`;
  ```
  The race here is straightforward: two CLI invocations call `docker ps -a` at the same time, both see no `${imageName}-0`, both pick `${imageName}-0`, and the second `docker run` fails with `Conflict. The container name "..." is already in use`. The new code never calls `docker ps -a` at all, so the race is structurally impossible.
- `packages/cli/src/utils/sandbox.test.ts:9` — adds `import { randomBytes } from 'node:crypto'`.
- `packages/cli/src/utils/sandbox.test.ts:34-36` — `vi.mock('node:crypto', () => ({ randomBytes: vi.fn().mockReturnValue(Buffer.from('a1b2c3d4e5f6', 'hex')) }))`. The fixed mock value lets the test pin the exact container name end-to-end.
- `packages/cli/src/utils/sandbox.test.ts:50-53` — removes the `if (cmd.includes('ps -a --format')) { return { stdout: 'existing-container', ... } }` branch from the exec mock. This is the proof that the test asserts no more `docker ps -a` call happens — the mock would have logged a stray invocation via the new `mockedExecCommands` accumulator.
- `packages/cli/src/utils/sandbox.test.ts:341-356` — new assertions for the regular path: `expect(mockedExecCommands).not.toEqual(expect.arrayContaining([expect.stringContaining('ps -a --format')]))`. This is the key invariant — the entire reason for the rewrite — captured as a single assertion.
- `packages/cli/src/utils/sandbox.test.ts:357-413` — new test `should preserve the integration-test prefix for random container names` exercises the `GEMINI_CLI_INTEGRATION_TEST=true` path and pins `gemini-cli-integration-test-a1b2c3d4e5f6` as the resulting name.

## Nits

1. The previous deterministic `${imageName}-0` / `${imageName}-1` naming was actually convenient for users who wanted to `docker exec -it gemini-cli-sandbox-0 sh` to debug. With random hex, that workflow now requires `docker ps` first. Consider keeping the deterministic name as the *display* name (the bit users see in logs / errors) while still using a random suffix on the container itself — i.e. log a friendly `gemini-cli-sandbox-0` alias even when Docker sees `gemini-cli-sandbox-a1b2c3d4e5f6`. Probably out of scope for this PR.
2. 6 bytes is fine but feels slightly arbitrary. Most container-orchestration tooling uses 8 chars (e.g. Docker's own short-id is 12 hex chars from a SHA-256). Either is fine; 6 bytes is consistent with the deleted `randomBytes(4)` integration-test code that the team was already comfortable with, just doubled.
3. The removal of the `ps -a` probe also removes an indirect health check on the Docker daemon (the previous code would have failed early if `docker ps` errored). Now the first command that touches Docker is `docker run`, which means daemon-down errors surface differently. Probably an improvement (fewer round-trips, simpler error path) but worth a smoke check on the existing daemon-down test, if any.

## What I learned

The "probe-then-pick" pattern for resource naming is a textbook TOCTOU bug, and it's particularly common with Docker because `docker ps` feels like a cheap, idempotent operation. The fix is always the same: replace the probe with sufficient entropy in the name itself. 48 bits (6 random bytes) is plenty for a CLI tool that creates at most a few containers per user per day; you'd need ~10^7 concurrent starts before collision probability hits 1%. The other useful pattern in this PR is the *negative* assertion in the test — `expect(mockedExecCommands).not.toEqual(expect.arrayContaining([expect.stringContaining('ps -a --format')]))` — which pins the *removal* of an unwanted side effect. That's much more durable than just asserting the new code path works; it prevents a future refactor from re-introducing the probe.
