# BerriAI/litellm PR #27241 — [Infra] Packaging: Relax Core Runtime Pins To Ranges

- URL: https://github.com/BerriAI/litellm/pull/27241
- Head SHA: `eff0f8c630b267f55ef1dbca15d05193422fbd2b`
- Size: +130 / -25 (one new workflow + pyproject.toml + uv.lock)

## Summary

Converts the 12 core runtime dependencies in `pyproject.toml` from exact pins (`==X.Y.Z`) to compatible-range constraints (`>=X.Y.Z,<NEXT_MAJOR`), and adds a new GitHub Actions workflow `check-dependency-floors.yml` that runs on every PR touching `pyproject.toml`, installs litellm at the *lowest-direct* resolution on Python 3.10 and 3.13, and smoke-imports the SDK plus the exact list of `openai`-namespace symbols litellm uses at runtime. `uv.lock` is regenerated to reflect the new specifiers.

## Specific findings

- `pyproject.toml:13-29` — 12 deps relaxed. Notable widenings: `openai==2.33.0` → `>=2.20.0,<3.0.0` (gives consumers a 13-minor-version corridor), `tiktoken==0.12.0` → `>=0.8.0,<1.0` (4 minors back), `aiohttp==3.13.4` → `>=3.10,<4.0` (3 minors back), `tokenizers==0.23.1` → `>=0.21.0,<1.0` (2 minors back). The 3-line comment block above the deps is load-bearing: explicitly tells future maintainers that Docker/CI reproducibility comes from `uv.lock` and the new floor-CI job validates the floors actually install — which is the right way to think about library-vs-application dependency policy.
- `.github/workflows/check-dependency-floors.yml:1-117` — new workflow. Triggered on `pull_request` only when `pyproject.toml` or this workflow itself changes (correct path filter), plus `workflow_dispatch` for ad-hoc runs. Matrix on Python 3.10 / 3.13 (the band litellm advertises). Pinned action SHAs (`actions/checkout@08eba0b…` v4.3.0, `astral-sh/setup-uv@37802adc…` v7) — supply-chain-safe. `permissions: contents: read` is minimum-necessary. `timeout-minutes: 10` reasonable.
- `.github/workflows/check-dependency-floors.yml:36-37` — `uv pip install --python /tmp/floor-venv/bin/python --resolution=lowest-direct .` is the load-bearing step. `lowest-direct` means *direct* deps go to their floor but transitive deps resolve normally — which is exactly what you want for testing "can a downstream consumer with the floor of openai actually import the SDK." If they had used `--resolution=lowest`, transitive deps would also floor and the test would over-constrain.
- `.github/workflows/check-dependency-floors.yml:48-83` — the smoke-import block is the real value. It imports `litellm`, then explicitly imports every openai-namespace symbol the codebase uses at runtime: `AsyncAzureOpenAI`, `AsyncOpenAI`, `AzureOpenAI`, `OpenAI`, `APIError`, `APITimeoutError`, `Omit`, plus the entire `openai.types.responses.*` surface (`ResponseFunctionToolCall`, `ResponseInputImageParam`, `ResponseOutputMessage`, `ResponseReasoningItem`, `IncompleteDetails`, `Response`, `ResponseOutputItem`, `Tool`, `ToolChoice`, `ResponseApplyPatchToolCall`, `ResponseTextConfigParam`, `FunctionToolParam`). This is the single most important defensive measure: a consumer who pins `openai==2.20.0` would silently install successfully today and then crash at runtime when litellm hits `from openai.types.responses.response_output_item import ResponseApplyPatchToolCall` (a symbol introduced after 2.20). The CI now catches that at PR time. **Comment at `:53-56` is honest and important:** "If a floor is set lower than the version that introduced any of these, the install will silently succeed and then fail at runtime in customer code. This list mirrors the openai imports under litellm/ — keep it in sync." The maintenance burden is real but explicit.
- `.github/workflows/check-dependency-floors.yml:69-82` — also imports the other 11 direct deps (`pydantic`, `httpx`, `aiohttp`, `tokenizers`, `tiktoken`, `click`, `jinja2`, `jsonschema`, `importlib_metadata`, `fastuuid`, `dotenv`). Coverage is shallow (just module-level import) but catches the most common breakage class.
- `.github/workflows/check-dependency-floors.yml:108-117` — `if: always()` resolved-versions report. Useful for triaging "this floor combination works on 3.10 but not 3.13" cases.
- `uv.lock:3263-3328` — regenerated `requires-dist` block. All 12 spec changes are reflected; transitive pins (a2a-sdk, anthropic, apscheduler, etc., all gated on `extra ==` markers) are untouched, which is correct — the relaxation policy is scoped to the *core* runtime, not optional extras.

## Notes

- The `openai>=2.20.0` floor is aggressive given the diff also keeps `ResponseApplyPatchToolCall` in the smoke-import (which I believe was added in a later 2.x). If the floor CI passes on 3.10 + 3.13, the floor is correct; if not, the floor needs to be raised. Reviewer should verify the CI run is green on this PR.
- `importlib-metadata>=8.0.0` (no upper bound) is the only unbounded spec. Justifiable since stdlib `importlib.metadata` is the long-term direction and `importlib-metadata` is the backport, but a `<10.0` cap would be defensive. Nit.
- No CHANGELOG entry. This is a packaging-policy change that downstream consumers (people building images on top of litellm) will care about — worth a release-notes paragraph.
- The "If a floor is set lower than the version that introduced any of these" comment block could link to a CONTRIBUTING.md section so future maintainers who add a new openai-symbol import know to update both sites.

## Verdict

`merge-after-nits`
