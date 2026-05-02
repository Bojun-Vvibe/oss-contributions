# QwenLM/qwen-code PR #3762 — feat(vscode): add message edit/rewind and message metadata UI

- Head SHA: `7d4e18cd26ba909faf2473c832a16e304eea3e9e`
- URL: https://github.com/QwenLM/qwen-code/pull/3762
- Size: +1513 / -49, 28 files
- Verdict: **needs-discussion**

## What changes

Adds a `rewindSession` extension method to the ACP agent surface and
wires the editor extension to call it. Key surface changes:

1. `packages/cli/src/acp-integration/acpAgent.ts:493-525` — new
   `case 'rewindSession'` in `extMethod`. Reads `sessionId`,
   `targetTurnIndex` from params, looks up the active session, calls
   `session.rewindToTurn(targetTurnIndex)`, returns
   `{ success, targetTurnIndex, apiTruncateIndex }`.
2. `packages/cli/src/acp-integration/acpAgent.test.ts:828-860` — new
   `rewindSession extension method rewinds the active session` test
   that mocks `Session.rewindToTurn` returning
   `{ targetTurnIndex: 1, apiTruncateIndex: 2 }`.
3. Session class gains `rewindToTurn(targetTurnIndex)` and the test
   harness now sets `innerConfig.getSessionId` to the session ID
   passed in.
4. The remaining ~1300 LOC adds the editor extension UI for message
   metadata display and the rewind button (not visible in the diff
   chunk reviewed but referenced in the PR description).

## What looks good

- Routing rewind through `extMethod` instead of inventing a new ACP
  method is the right call — keeps the protocol surface stable while
  extension capabilities evolve.
- The test mocks the right boundary (`Session.rewindToTurn`) and
  asserts both the call argument and the return shape.
- `lastSessionMock` capture pattern lets the test reach into the
  mocked session without exposing it globally.

## Concerns / discussion needed

### 1. Authorization / multi-session guardrails

The `rewindSession` handler accepts a `sessionId` param and acts on
"the active session". From the diff:

```ts
const sessionId = params['sessionId'] as string;
const targetTurnIndex = params['targetTurnIndex'];
```

Questions for the author:
- Does the handler verify that `sessionId` matches the session this
  ACP connection was bound to? An editor with multiple concurrent
  sessions could rewind the wrong one if the param isn't validated.
- What happens if `targetTurnIndex` is out of range, negative, or a
  non-integer? The diff snippet does no validation — the test only
  exercises `targetTurnIndex: 1` happy path.

### 2. Type safety on params

`params['sessionId'] as string` and `params['targetTurnIndex']` (no
cast) bypass the TS type checker. ACP extension methods are inherently
dynamic, but this entry point should at least:
- Validate `typeof sessionId === 'string'`
- Validate `Number.isInteger(targetTurnIndex) && targetTurnIndex >= 0`
- Return a typed error response (e.g. `{ success: false, error: '...' }`)
  on failure, not throw.

### 3. Side effects of rewind on persisted state

Rewinding a session implies truncating the message log. The test
returns `apiTruncateIndex: 2` — what is the contract here? Does the
caller persist the truncation, or is it advisory? If a rewind is
issued and then the editor crashes, what state is the session in on
restart? Worth a short design note in the PR body.

### 4. PR scope

This is a 1513-line PR touching 28 files. The extension method itself
is small and clean; the surrounding metadata-UI work is large and
unrelated. Splitting into:
- (a) `feat(core): add Session.rewindToTurn`
- (b) `feat(acp): expose rewindSession extension method` (this PR's
      ACP slice)
- (c) `feat(editor-ext): message metadata UI + rewind button`

would make each piece reviewable on its own merits.

## Risk

Medium-high if landed as one unit due to scope. The ACP slice itself
is low-risk pending the validation gaps above.
