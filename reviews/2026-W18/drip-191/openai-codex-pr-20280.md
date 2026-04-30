# openai/codex #20280 — Use powershell AST parser in exec_command for prefix-rule matching

- **URL:** https://github.com/openai/codex/pull/20280
- **Head SHA:** `57a1d792afbd3de6a6e18b4c9cc1efa70c3affc4`
- **Files:** `codex-rs/core/src/exec_policy.rs` (+39/-13), `codex-rs/core/src/exec_policy_tests.rs` (+181/-0), `codex-rs/core/tests/suite/exec_policy.rs` (-5), `codex-rs/shell-command/src/command_safety/mod.rs` (+1), `codex-rs/shell-command/src/command_safety/powershell_parser.rs` (+10/-0), `codex-rs/shell-command/src/command_safety/windows_safe_commands.rs` (+12/-128), `codex-rs/shell-command/src/powershell.rs` (+228/-1)
- **Verdict:** `merge-as-is`

## What changed

Pulls the previously command-safety-private PowerShell wrapper-parsing logic up into a public, dual-mode parser used by *both* the safety check and the exec-policy prefix-rule evaluator on Windows.

Three load-bearing pieces:

1. **New `PowershellCommandSequenceParseMode` enum** at `shell-command/src/powershell.rs:579-583`:
   ```rust
   pub enum PowershellCommandSequenceParseMode {
       ExecPolicy,
       SafeCommand,
   }
   ```
   This distinguishes "I'm asking 'is this safe to auto-approve' (strict — reject any wrapper flag I don't know is benign)" from "I'm asking 'what inner command does this powershell wrapper actually invoke, for prefix-rule matching' (lenient — skip wrapper flags I don't recognize so I can still see the inner command)".

2. **Mode-aware `parse_powershell_invocation`** at `:610-682`. The matrix:
   - `-nologo`/`-noprofile`/`-noninteractive`/`-mta`/`-sta` (`POWERSHELL_NO_ARG_PARSE_FLAGS` at `:574-575`): always tolerated.
   - `-windowstyle`/`-executionpolicy`/`-workingdirectory` (`POWERSHELL_VALUE_PARSE_FLAGS` at `:576-577`): in `SafeCommand` mode → reject (`return None`); in `ExecPolicy` mode → consume `idx += 2` and continue.
   - `-encodedcommand`/`-ec`/`-file`/`/file`: always reject (opaque payloads).
   - Unknown `-foo` / `/foo` flags: in `SafeCommand` → reject; in `ExecPolicy` → call `powershell_wrapper_flag_length` at `:789-799` to consume one or two tokens depending on whether the next token looks like a flag, then continue.
   - Bare `-Command <script>` / `-c <script>` / `-Command:<script>`: parse the script via `parse_powershell_script_to_commands` at `:684-689` which calls the new `try_parse_powershell_ast_commands` helper at `command_safety/powershell_parser.rs:378-386`.

3. **`exec_policy.rs` integration at `:243-307`**. The `exec_policy_for_command` flow on Windows:
   - Calls `try_parse_powershell_command_sequence(original_command, ExecPolicy)` at `:244-249`.
   - If parsing succeeded, `exec_policy.check_multiple_with_options` at `:294-298` evaluates the inner commands.
   - **Outer-policy reconciliation** at `:303-330` is the security-critical step: independently calls `exec_policy.matches_for_command_with_options(original_command, /*heuristics_fallback*/ None, ...)` at `:307-311`. If the **outer wrapper** matches an explicit policy rule (e.g. `prefix_rule(pattern=["powershell.exe"], decision="forbidden")`), drop all heuristic-only matches at `:319-321`, append the outer-rule matches, and recompute `evaluation.decision` as the *max* (in the `Forbidden > Prompt > Allow` ordering) of all surviving rules at `:322-329`. This is the test asserted at `exec_policy_tests.rs:241-265` (`powershell_wrapper_rules_still_apply_when_inner_commands_are_parsed`).
   - **Heuristics-fallback override** at `:288-294`: when the inner commands hit the heuristics fallback (no rule matched), substitute `original_command` back into the `RuleMatch::HeuristicsRuleMatch.command` field so the operator sees the user's actual invocation in error messages and prompts, not the parsed inner form.

A separate trust-control gate at `trusted_powershell_parser_executable` (`:691-705`): when the user invokes `C:\repo\powershell.exe` (untrusted path), the AST parser uses `try_find_powershell_executable_blocking()` to resolve the *system* `powershell.exe`, not whatever's on PATH. The test at `:843-862` (`uses_trusted_system_powershell_for_ast_parsing`) locks this — if a malicious user drops a trojaned `powershell.exe` in their cwd and invokes `.\powershell.exe -Command 'Remove-Item ~/.bashrc'`, the parser still uses the trusted binary to AST-parse the script, so the prefix rule on `Remove-Item` still fires.

The bypass-sandbox shortcut at `exec_policy.rs:344-350` is also tightened: previously, `bypass_sandbox = commands.iter().all(|cmd| <re-evaluate per-command>)` re-ran the policy match per command. New shape collapses to `evaluation.matched_rules.iter().all(|rule_match| is_policy_match(rule_match) && rule_match.decision() == Decision::Allow)` — single pass over the unified matched_rules set, including the outer-policy matches added at `:319-321`. This means a `powershell.exe -Command 'Get-Content x'` where `Get-Content` is allowed but `powershell.exe` itself isn't matched (heuristic-only) won't bypass sandbox — exactly the right shape.

## Why it's right

- **Closes a real evasion class.** Previously, `prefix_rule(pattern=["git", "push"], decision="forbidden")` would block `git push` directly but *not* `powershell.exe -Command 'git push origin main'` because the policy evaluator only saw `["powershell.exe", "-Command", "..."]` as the prefix. The five test cases at `exec_policy_tests.rs:160-237` lock the new behavior across `-WindowStyle Hidden`, `-ExecutionPolicy Bypass`, `-WorkingDirectory C:\repo`, `-Version 5.1 -NoExit`, and bare wrappers — all of which previously bypassed the prefix rule.
- **Outer-rule reconciliation at `:303-330` is the right shape**: the wrapper-was-parsed case doesn't *only* match the inner commands. If `powershell.exe` is itself forbidden (or the user has a policy that says "anything that goes through powershell needs approval regardless of payload"), that decision still wins via the `evaluation.decision = ...max()` at `:323-329`. Test at `:241-265` proves it.
- **The dual-mode design is the right factoring.** The old `windows_safe_commands.rs` had its *own* private wrapper parser at `:124-128` (deleted) that was strict-by-construction (rejecting any unknown flag). Reusing the same module from exec-policy would have either weakened the safe-command check (bad) or rejected too many exec-policy parses (also bad). The mode parameter at the caller boundary makes the policy explicit per call site rather than a flag inside the parser. The new test at `windows_safe_commands.rs:551-557` locks that wrapper flags exec-policy tolerates stay unsafe for auto-approve.
- **`trusted_powershell_parser_executable` at `:691-705` closes the trojaned-shim attack.** The PowerShell AST parser is invoked by spawning `powershell.exe -Command "{$null = [...]}"` or similar — if we use the user's invoked path, a malicious binary at `.\powershell.exe` could simply emit a benign AST regardless of the actual script. Using the system `powershell.exe`/`pwsh.exe` from PATH for parsing keeps the trust assumption intact. Worth calling out: if PATH itself is poisoned, we lose, but that's a strictly bigger compromise than this PR can mitigate.
- **The TODO removal at `core/tests/suite/exec_policy.rs:81-86`** (`if cfg!(windows) { return Ok(()); }`) shows the test was previously skipped on Windows because "execpolicy doesn't parse powershell commands yet" — this PR is the fix that lets that test pass on Windows.
- **`unmatched_powershell_wrappers_keep_outer_command_heuristics`** at `:268-316` is the load-bearing case for "wrapper parses succeed but no rule matches": the inner-command heuristic still uses the *outer* `powershell.exe -Command 'Get-Content Cargo.toml'` as the displayed command and the proposed amendment, so prompts show the user's actual command. This is the substitution at `:317-321`.

## Nits / not blockers

- The `_ if is_unsupported_powershell_parse_flag(&lower) || has_unsupported_powershell_parse_flag_inline_value(&lower)` arm at `:657-661` doesn't distinguish between `SafeCommand` and `ExecPolicy` modes — `-EncodedCommand` is always rejected. That's correct (we can't AST-parse base64 without decoding it, and decoding-then-recursing-into-policy-eval invites a parser-confusion attack), but the current behavior means `powershell.exe -EncodedCommand <b64>` falls through to the heuristic prompt path. An explicit "this command was rejected because the wrapper is opaque" hint would help operators know they need to inline the command instead of base64-encoding it.
- `powershell_wrapper_flag_length` at `:789-799` uses `looks_like_powershell_flag(&next_arg.to_ascii_lowercase())` to decide one-vs-two-token consumption. For a flag like `-NoExit` followed by another flag that's the right call (`-NoExit` takes no arg). For an unknown vendor flag like `-MyFancyFlag value`, it consumes both. This is the lenient choice — the strict alternative (always consume 1) would mean unknown flag values get treated as the script body. Document the heuristic with a one-line comment naming the trade-off.
- `is_powershell_value_parse_flag` at `:751-757` uses `POWERSHELL_VALUE_PARSE_FLAGS.contains(&lower)` then a separate `matches!` for the `/`-prefixed forms. Either fold them into one constant table or extract a helper — currently there are three slightly-different ways the file matches flag names (`POWERSHELL_NO_ARG_PARSE_FLAGS`, `POWERSHELL_VALUE_PARSE_FLAGS`, the inline-value `matches!`). One canonical normalize-and-lookup would be easier to keep in sync as PowerShell adds new wrapper flags.

## Risk

Low. This is purely a Windows-path policy-evaluator surface; non-Windows `try_parse_powershell_command_sequence` returns `None` via the `#[cfg(not(windows))]` branch at `exec_policy.rs:251-252` (`let powershell_commands: Option<Vec<Vec<String>>> = None`). The new module-level `pub` exports on `powershell.rs` are additive. The test coverage at `exec_policy_tests.rs:160-339` plus the surface-test at `powershell.rs:820-862` covers the five-tuple of (rule outcome × wrapper variant) plus the trusted-parser-resolution path. The previously-skipped Windows integration test at `tests/suite/exec_policy.rs:81-86` will now run; if that test reveals a regression, it'll show up before merge.
