# QwenLM/qwen-code PR #3797 — feat(cli): add /model list subcommand for dynamic model discovery

- Head SHA: `99cb79637239bf63e6ca021338a8bfc0a4fad333`
- URL: https://github.com/QwenLM/qwen-code/pull/3797
- Size: +99 / -0, 1 file (`packages/cli/src/ui/commands/modelCommand.ts`)
- Verdict: **request-changes**

## What changes

Adds a `list` subcommand under the existing `/model` slash command.
When invoked, it reads `baseUrl` and `apiKey` from the active
`contentGeneratorConfig`, hits `${baseUrl}/models` with `Authorization:
Bearer <apiKey>`, and prints one model id per line. Existing `/model`,
`/model <id>`, and `/model --fast` behavior is preserved — the new
subcommand is grafted into `subCommands: [...]` on the existing
`modelCommand` export. Closes #2412.

## What looks good

- The choice to hang this off the existing `/model` command rather
  than adding `/models` as a top-level command keeps the slash
  namespace tidy. Discoverability via `/model list` is consistent
  with how `/session list` etc. work elsewhere.
- `supportedModes: ['interactive', 'non_interactive', 'acp']` (line
  ~155) is the full set, which is correct: `qwen -p "/model list"`
  in non-interactive should work, and the PR body explicitly
  validates this with Groq + Hugging Face.
- The trailing-slash normalization `baseUrl.replace(/\/+$/, '')`
  (line ~232) handles both `https://api.openai.com/v1` and
  `https://api.openai.com/v1/` cleanly, which is a real footgun for
  this kind of glue.
- Type narrowing on the response: the `.filter((id): id is string =>
  typeof id === 'string' && id.length > 0)` (line ~256) is the right
  defensive shape for an OpenAI-compatible-ish endpoint that may
  return malformed entries.

## Why request-changes

1. **Silent leak of API key in error path.** `fetchModels` (line
   ~243) does `throw new Error(\`Request failed (${response.status}):
   ${errorText}\`)`. `errorText = await response.text()` is the raw
   body of the failed response. Many OpenAI-compatible providers
   echo back the request authorization header or the api_key query
   param in their 4xx error bodies (Hugging Face does this in some
   cases). The error message then surfaces in the chat transcript
   via `messageType: 'error'`. This needs to either (a) cap
   `errorText` at e.g. 500 chars *and* run a redaction pass that
   strips anything that looks like a bearer token, or (b) only
   include `response.status` + `response.statusText` and log the
   body to debug logs only.
2. **No timeout / no AbortController.** A misconfigured `baseUrl`
   pointing at a black hole will hang the slash command indefinitely.
   `fetch(url, { method: 'GET', headers })` (line ~241) needs a
   `signal: AbortSignal.timeout(10_000)` or equivalent. Without it,
   in interactive mode the user has no way to cancel except killing
   the whole CLI.
3. **Output is unbounded.** Some providers (Hugging Face inference,
   Together) return *thousands* of model IDs. `models.join('\n')`
   into a single `messageType: 'info'` content block will dump them
   all at once with no pagination. At minimum, cap at 200 rows and
   append `(showing 200 of N — use \`/model list <prefix>\`)`. Adding
   the prefix-filter is out of scope per the PR body, but the cap
   isn't.
4. **No test coverage.** A new fetch-from-network slash command with
   auth-header injection and zero unit tests. At minimum:
   `fetchModels` should be testable via a mock fetch (URL
   normalization, auth header presence, malformed `data` handling,
   non-2xx error shape). This file's existing surface is testable;
   adding tests is straightforward.
5. **`baseUrl` from `contentGeneratorConfig` may be undefined for
   some provider configs.** The `if (!baseUrl)` guard (line ~190)
   handles this, but the error message says "Please configure
   modelProviders or set the API endpoint" — that text doesn't match
   the actual config knob names a user would search for. Match the
   exact key from the config schema.

## Nits

1. The `subCommands` array is appended *after* the `action` property
   in the parent command (line ~152). Object property order in TS
   doesn't matter for runtime, but stylistically `subCommands`
   typically goes adjacent to `name` / `description`.
2. The `Accept: 'application/json'` header is sent but no
   `Content-Type` (correct for GET, just confirming). No
   `User-Agent` either — some providers rate-limit by missing UA.

## Risk

Medium. The auth-bleed-on-error and infinite-hang issues are both
realistic failure modes that will surface in user bug reports, not
just edge cases. The functional happy path is fine; the failure
paths need hardening before merge.
