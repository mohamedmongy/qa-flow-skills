# QA Flow — Step Type: `wait_until`

Repeatedly calls an API or saved query until a condition is satisfied or the timeout expires. Raises `WaitUntilTimeoutError` if the condition is never met within `timeout` seconds.

## Variant 1 — Poll an API

```json
{
  "type": "wait_until",
  "name": "step_2",
  "api": "get_order_status",
  "test_case": "default",
  "timeout": 60,
  "interval": 5,
  "condition_mode": "visual",
  "condition": {
    "field": "data.status",
    "operator": "eq",
    "value": "completed"
  },
  "context_export": ["data.status"],
  "response_export": ["data.status"],
  "header_import": []
}
```

## Variant 2 — Poll a DB Query

```json
{
  "type": "wait_until",
  "name": "step_3",
  "query_method": "get_order_by_id",
  "query_params": { "order_id": "{{context.step_1.orderId}}" },
  "timeout": 30,
  "interval": 10,
  "condition_mode": "visual",
  "condition": {
    "field": "result.status",
    "operator": "contains",
    "value": "refund"
  }
}
```

**Important:** DB query results are wrapped under `result`. Always use `result.<field>` in the condition — never bare field names.

## Variant 3 — Python Code Condition

```json
{
  "type": "wait_until",
  "name": "step_3",
  "api": "poll_job",
  "test_case": "default",
  "timeout": 120,
  "interval": 10,
  "condition_mode": "python_code",
  "python_code": "status = result.get('data', {}).get('status')\nreturn status == 'done'"
}
```

## Field Reference

| Field | Required | Notes |
|---|---|---|
| `api` + `test_case` | one pair required | API polling variant |
| `test_cases` | no (instead of `test_case`) | Array of test case names — each one is polled in turn, with its own full `timeout` window |
| `query_method` + `query_params` | one pair required | DB polling variant |
| `timeout` | yes | Max seconds before timeout error |
| `interval` | yes | Seconds between polling attempts |
| `condition_mode` | yes | `"visual"` or `"python_code"` |
| `condition` | if visual mode | `{ field, operator, value }` |
| `python_code` | if python_code mode | Must `return True` (done) or `return False` (keep waiting) |
| `context_export` / `response_export` | no | Fields to save from the final successful response |
| `header_import` | no | Same as `api_call` — injects context values as headers |

## Multiple Test Cases

Like `api_call`, a `wait_until` step may select several test cases (`test_cases`). Each one is polled against the same condition **in array order**, each with its own full `timeout`/`interval` budget and its own payload/params.

**Order matters:** exports follow the same **last wins** rule — only the last polled test case's response reaches `<step>.<field>`, so list the test case whose response later steps consume last. The step is marked **failed** if any test case times out or errors (the remaining test cases still run).

Note the worst-case duration: N test cases × `timeout` seconds. Keep it inside the flow's `expected_flow_execution_time`.

## Visual Mode Condition Operators

`wait_until` conditions are evaluated by `ConditionEvaluator` (`Common/condition_evaluator.py`), which accepts a **different operator set than `conditional` steps**. Using an unsupported operator raises `ValueError: Unsupported operator '...'` at runtime.

| Operator | Meaning |
|---|---|
| `eq` | Exact equality |
| `ne` | Not equal |
| `gt` | Greater than |
| `gte` | Greater than or equal |
| `lt` | Less than |
| `lte` | Less than or equal |
| `contains` | Substring / membership match |
| `not_contains` | Negated substring / membership match |

> ⚠️ **Do not use `==`, `!=`, `>`, `<`, `is_empty`, or `is_not_empty` here** — those are `conditional`-step operators and will raise `ValueError` in a `wait_until` condition. Use `eq`, `ne`, `gt`, `lt` instead. The full operator list lives in `SUPPORTED_OPERATORS` in `Common/condition_evaluator.py`.

String values are coerced to match the actual value's type before comparison (e.g. `"true"`/`"1"`/`"yes"` → `True`, numeric strings → `int`/`float`), so a `value` declared as a string still compares correctly against booleans and numbers.

## Python Code Mode — Available Variables and Helpers

The snippet runs with the full helper environment built by `build_exec_globals` (`code_generator/generate_test_suite/helpers/flow_context.py`), re-evaluated on every poll attempt. Available names:

- `result` — the parsed response or DB record from the **current** poll attempt (injected fresh each poll)
- `get_var("step_N.fieldName", default=None)` — read any flow context value
- `set_var(name, value)` — export a value for later steps
- `get_env(name, default=None)` — read an environment variable
- `parse_json(text)` / `to_json(obj, pretty=False)` — JSON helpers
- `get_response(path, default=None)` — read from the last response
- `mask(value, visible=4)` — mask a secret for logging
- `now(fmt)`, `uuid()`, `sleep(seconds)`, `retry(fn, times=3, delay=1.0)`
- `assert_that(expr, message)` / `expect(actual, op, expected, msg)` — inline assertions
- `log(message, level)` — log a message (`"info"`, `"warning"`, `"error"`) and `dump_context()` for debugging
- Any **user helpers** under `user_data/user_helpers/*.py` — both flat function names and module-namespaced (e.g. `token_utils.mask_token(...)`) — but only if those files actually define them

The code must `return True` to signal the condition is met (stop polling) or `return False` to keep waiting (the runner wraps the return in `bool(...)`).

## header_import in wait_until

`header_import` works identically to `api_call` steps. Use it to inject tokens or tenant headers needed by the polled API:

```json
"header_import": [
  { "header": "Authorization", "prefix": "Bearer ", "variable": "login_step.accessToken" }
]
```
