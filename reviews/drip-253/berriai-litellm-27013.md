# BerriAI/litellm #27013 — fix(prometheus): escape api_key for PromQL string literal (VERIA-53)

- **Repo:** BerriAI/litellm
- **PR:** https://github.com/BerriAI/litellm/pull/27013
- **HEAD SHA:** `6a5ecafdffa1cc3184c2fb72f03399f9063cd45d`
- **Author:** stuxf
- **Verdict:** `merge-after-nits`

## What the diff does

Closes a PromQL injection at `get_daily_spend_from_prometheus` where a
caller-supplied `api_key` was f-string-interpolated directly into a
label-matcher value:

```python
query = f'sum(delta(litellm_spend_metric_total{{hashed_api_key="{api_key}"}}[1d]))'
```

A bare `"` in `api_key` would terminate the matcher and let the
attacker append arbitrary PromQL — `sk-victim"} or
sum(other_metric{a="b` rendered as
`...hashed_api_key="sk-victim"} or sum(other_metric{a="b"}[1d]))` is
two valid PromQL expressions ORed together, returning cross-tenant
spend metrics.

Two-file fix:

1. `litellm/integrations/prometheus_helpers/prometheus_api.py:85-99`
   — new `_quote_promql_string_literal(value: str) -> str` returns
   `json.dumps(value, ensure_ascii=False)`, with a docstring explaining
   that PromQL string literals follow Go's escape rules and JSON's
   quoting is a strict subset of Go's, so `json.dumps` produces a
   literal Prometheus parses identically.
2. Same file, `:128-135` — replaces the f-string interpolation with
   `quoted_api_key = _quote_promql_string_literal(api_key)` and a
   three-piece string assembly that places `quoted_api_key` as the
   complete RHS of the label matcher (including the surrounding
   double quotes that `json.dumps` emits).

Six tests at `tests/test_litellm/integrations/test_prometheus_api_promql_escape.py`:

- `test_quote_safe_input_round_trips` (`:14-22`) — pin the no-op
  case so a future "let's add slug-style escaping" change doesn't
  silently mangle legitimate keys.
- `test_quote_escapes_double_quote` (`:24-32`) — the load-bearing
  injection-prevention assertion.
- `test_quote_escapes_backslash` (`:34-40`) — closes the
  `\` → `\\"` two-step escape (the actual mechanism JSON uses).
- `test_quote_escapes_newlines_and_control_chars` (`:42-54`) —
  documents the `\n` / `\t` / `\r` escape behavior beyond the
  security minimum.
- `test_get_daily_spend_does_not_pass_raw_quote_into_query`
  (`:57-95`) — end-to-end injection test that asserts the rendered
  query contains `\"` for every injected `"` and the matcher
  framing (`...hashed_api_key="`...`"}[1d]))`) is intact.
- `test_get_daily_spend_with_no_api_key_uses_unfiltered_query`
  (`:98-117`) — locks the `api_key is None` branch unchanged.
- `test_get_daily_spend_legitimate_hashed_key_unchanged`
  (`:120-148`) — locks that 64-char hex hashed keys flow through as
  themselves, no spurious escaping.

## Why it's right

The chosen primitive (`json.dumps`) is the right one. PromQL's string
literal grammar (per
https://prometheus.io/docs/prometheus/latest/querying/basics/) is a
proper superset of JSON's — every valid JSON string literal is also a
valid PromQL string literal with the same interpretation. So the
delegation is sound and doesn't require the author to maintain their
own escape table. The docstring at `prometheus_api.py:88-99` calls this
out explicitly, which is the right thing to write down because someone
reading "we use `json.dumps` for SQL... wait, no, PromQL" five months
from now will need that breadcrumb.

The end-to-end test at `:57-95` is the dispositive piece. It doesn't
just check that `_quote_promql_string_literal` escapes the quote; it
checks that the *rendered query* (after the call goes through the full
function) cannot be parsed as an injection — by stripping `\"`
sequences and asserting no bare `"` remains in the inner matcher
value. That assertion shape is robust against future refactors that
change the assembly mechanism (string concatenation vs. f-string vs.
template) as long as the security property holds.

The `model_specific_request` and other f-string call sites on
`prometheus_api.py` (none in this diff that I can see, but worth a
grep — see nit 2) are the obvious follow-up surface. The fix is
correctly scoped to the immediate VERIA-53 finding.

## Nits

1. **`ensure_ascii=False`** at `:99` is a defensible choice (preserves
   Unicode in label values rather than `\uNNNN`-escaping it) but
   PromQL's parser tolerates both. The reason `False` is right is
   that a hashed_api_key is hex so this never fires for the
   load-bearing case, and for diagnostic queries with non-ASCII
   labels (rare) it preserves readability. Worth one line in the
   docstring noting this — otherwise a future "let's normalize all
   PromQL strings to ASCII" change might flip it without realizing
   it's intentional.

2. **No grep for sibling f-string PromQL injections** in
   `prometheus_api.py` and adjacent helpers. The PR title says
   "VERIA-53" suggesting a single-finding scope, but if the
   security audit found this one by code review, the same pattern
   may exist for other label matchers (`team_id`, `model_id`,
   `user_id`, etc.). Worth confirming in the PR description that a
   `grep -nE 'f["'\'']?.*\{[a-z_]+="\{' litellm/integrations/`
   pass was done — if not, this fix closes one site but leaves
   others open.

3. **Naming**: `_quote_promql_string_literal` is accurate but the
   leading underscore signals "private to this module" which is
   right for now; consider promoting to a shared helper if any
   other module also constructs PromQL queries (the same audit
   probably wants to apply this to any other prom-query-building
   site).

4. **Test comment at `:71-73`** says "The legitimate matcher framing
   must still be intact" — worth strengthening that to "and the
   subsequent `}[1d]))` is *not* split off into a separate PromQL
   expression by an unescaped quote", which is the actual
   security-relevant assertion. The current assertion is correct
   but the comment undersells what it proves.

5. **`json.dumps` will raise `TypeError` on non-str input**
   (e.g. an integer). The function signature says `value: str` but
   Python doesn't enforce that at runtime. If `api_key` is ever
   `None` (the caller already short-circuits), or an int via a
   buggy upstream path, the failure mode is a 500 with stack trace
   rather than a quiet rejection. A defensive
   `if not isinstance(value, str): raise ValueError(...)` at the
   top of the helper would make the contract explicit.

## Verdict rationale

Right primitive (`json.dumps` is provably-correct PromQL escaping for
strings), right shape of fix (helper + replace one call site), right
test surface (unit assertions on the helper + end-to-end on the
rendered query). Documentation nit on `ensure_ascii`, recommend a grep
for sibling injection sites, and a defensive type guard on the helper
input.

`merge-after-nits`
