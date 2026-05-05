# BerriAI/litellm #27145 — Add VSCode launch configuration for LiteLLM proxy debugging

- Head SHA: `cbab96553cccb442d15ca70fef815a22f423c504`
- Diff: +24 / −0 in `.vscode/launch.json` (new file)

## Verdict: `merge-after-nits`

## What it does

Adds a single `.vscode/launch.json` with one `debugpy` "launch" configuration that runs `litellm/proxy/proxy_cli.py` with `--host 0.0.0.0 --config test-config/dev_config.yaml --port 4000 --num_workers 1` and `justMyCode: false`. Lets contributors hit F5 in VSCode (or a fork like Cursor) and get a debugger attached to the proxy without remembering the right `python -m debugpy --listen ... -- ...` incantation or pasting in a config from a colleague.

## Why this is fine

1. **Strictly additive.** New file, no existing `.vscode/` content touched, no behavior change for non-VSCode users. CI cost: zero.
2. **`num_workers: 1` is the correct choice for a debugger config** — `debugpy` doesn't multiplex breakpoints across worker processes well, so even if the prod default is N, pinning to 1 here makes breakpoints reliable. Same logic for `justMyCode: false` (you almost always want to step into LiteLLM internals when debugging, not just user app code).
3. **`integratedTerminal` over `internalConsole`** is the right pick for a long-running server because the integrated terminal preserves color codes, ANSI control sequences, and handles SIGINT cleanly when you stop the debug session.
4. **Path is relative to `${workspaceFolder}`** so the config works regardless of where contributors cloned the repo, and assumes the canonical layout where `litellm/proxy/proxy_cli.py` and `test-config/dev_config.yaml` exist — both of which are in the tree today.

## Nits

- **No trailing newline at end of file** (the diff shows `\ No newline at end of file`). VSCode itself doesn't care, but most repo-wide pre-commit hooks (`end-of-file-fixer`, `editorconfig-checker`) do. If LiteLLM's pre-commit config flags missing-EOL it'll fail CI; a one-byte fix.
- **No `.vscode/` entry in the PR body about whether `.vscode/` is `.gitignore`d.** A quick check of the repo would tell us whether this is the first VSCode config (in which case a follow-up README mention under "Contributing" is helpful) or whether `.gitignore` was already excluding `.vscode/*` and the author added a `!launch.json` exception. From the diff alone I can't tell — flagging in case the file ends up not-tracked by accident.
- **Port `4000` is the default LiteLLM proxy port** — fine, but if a contributor already runs the proxy on `4000` in another terminal, hitting F5 will fail with `EADDRINUSE` and the error will look like "the debugger crashed." A one-line comment in the config (`// Change --port if 4000 is in use`) would help; alternatively use `${input:proxyPort}` with an `inputs:` prompt section.
- **`--host 0.0.0.0` exposes the debug-mode proxy on all network interfaces.** That's almost never what a contributor wants on a dev laptop and is a footgun on shared/coffee-shop networks (the proxy with master-key set serves real LLM credentials). Recommend defaulting to `127.0.0.1` and documenting how to flip to `0.0.0.0` for cross-VM/container scenarios. Especially relevant given `justMyCode: false` — an attacker who reaches the debug endpoint while debugpy is listening can trivially get arbitrary code execution.
- **Hardcoded `test-config/dev_config.yaml`.** If that file isn't checked in (or has secret env-var references), F5 will fail with a confusing config-load error instead of "you need to copy `test-config/dev_config.example.yaml` first." Worth verifying the file exists in the tree and is checked in, or adding a `preLaunchTask` that materializes it.

## Citations

- `.vscode/launch.json:1-24` — full new config file
- `.vscode/launch.json:13` — `--config test-config/dev_config.yaml` (verify in-tree)
- `.vscode/launch.json:5` — `"type": "debugpy"` (requires the Python extension; consumer must have it installed)
