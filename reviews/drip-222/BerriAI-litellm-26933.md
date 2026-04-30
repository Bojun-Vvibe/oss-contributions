---
pr-url: https://github.com/BerriAI/litellm/pull/26933
sha: f52e29c66a00
verdict: merge-as-is
---

# fixed circleci syntax

Two-line CI hotfix at `.circleci/config.yml:1483` and `:1490`: the heredoc redirection `cat > test-results/junit.xml << EOF` is escaped to `cat > test-results/junit.xml \<<EOF` in both arms (success and failure) of the `claude_code_gateway_e2e` test result emission. The PR title and absence of body says it all — this is fixing a CircleCI YAML-shell-quoting parser issue introduced when the surrounding shell-script block was wrapped in some flavor of CircleCI templating that consumed the unescaped `<<` as a YAML node anchor or template token.

The fix is correct in shape: in CircleCI's `run:` step, the multi-line `command:` is YAML-parsed first and then handed to bash, and the unescaped `<<` is the CircleCI parameter-substitution syntax (`<< parameters.foo >>` / `<< pipeline.git.branch >>`). Bare `<<EOF` immediately after a `cat >` redirection is ambiguous to the YAML preprocessor — it can be read as either the start of a heredoc (intended) or the start of a CircleCI substitution (which then complains about the missing `>>` close). Backslash-escaping the first `<` (`\<<EOF`) defeats the substitution-parser, the backslash gets eaten by bash before the heredoc operator is parsed, and the result is the intended `cat > file <<EOF ... EOF` heredoc.

Both occurrences in the diff are inside an `if/else` over the gateway-test exit code — the success arm emits a passing `<testcase>`, the failure arm emits a failing `<testcase>` with the captured `$ESCAPED_OUTPUT` as the failure message. Both arms previously had the same syntax bug and both arms get the same one-character fix, which is the right symmetric shape — fixing only one arm would have left the other broken under the opposite test outcome.

The trade-off is that `\<<EOF` reads as "what's that backslash doing there?" to anyone unfamiliar with CircleCI's parameter-substitution syntax. A defensive comment above the first occurrence ("# escape `<<` to defeat CircleCI parameter-substitution preprocessor; bash eats the backslash before parsing the heredoc") would have helped the next contributor who looks at this and tries to "clean up" the unnecessary-looking escape. But that's a nit-level improvement, not a blocker — the fix itself is correct, minimal, and symmetric.

No tests required (or possible — this is a CircleCI-config-only change that exercises the parser, and the only meaningful "test" is "the next CI run doesn't fail with a YAML/template parse error"). The author presumably observed the CI break, applied the escape, watched CI go green, and shipped — that's the right workflow for a config hotfix.

The PR is a one-line followup to whatever change introduced the surrounding template-syntax sensitivity (likely a CircleCI orb upgrade or a `parameters:` block addition on this job). The lack of body or context-PR-link is the only nit, and it's small enough not to gate merge — the diff is self-explanatory once you know the CircleCI parameter-substitution rule.

## what I learned
CI-config heredoc-quoting bugs are the silent killer of "I'll just paste a multi-line script into the YAML" workflows because every CI provider has its own templating preprocessor that runs before bash gets the string. CircleCI uses `<< >>`, GitHub Actions uses `${{ }}`, Azure Pipelines uses `$( )` and `$[ ]`, GitLab CI uses `$VAR` directly — and each one has different escape rules for the corresponding syntax. The defensive default for any multi-line shell block in a CI YAML is "escape the provider's templating syntax," and the load-bearing knowledge is which provider's templating runs. The right preventative shape for litellm specifically would be a shellcheck-or-equivalent CI lint that catches unescaped `<<` adjacent to a heredoc operator in `.circleci/config.yml`, but adding that lint is a bigger project than this PR.
