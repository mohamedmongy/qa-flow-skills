# QA Flow — Step Types: `conditional` and `flow`

---

## `conditional` — Branch on Context Values

Evaluates conditions against flow context and takes one of two actions.

### Visual Mode

```json
{
  "type": "conditional",
  "name": "step_5",
  "condition_mode": "visual",
  "conditions": [
    { "field": "step_2.accessToken", "operator": "is_not_empty", "value": "" }
  ],
  "logic_operator": "AND",
  "on_true": {
    "action": "continue",
    "message": "Token found, proceeding"
  },
  "on_false": {
    "action": "warn",
    "message": "Token missing — check login step"
  }
}
```

### Python Code Mode

```json
{
  "type": "conditional",
  "name": "step_6",
  "condition_mode": "python_code",
  "python_code": "quota = get_var('step_4.result.quota_value')\nif quota is not None and int(quota) > 0:\n    return continue_flow()\nelse:\n    return raise_error()"
}
```

### Visual Mode Condition Operators

`==`, `!=`, `>`, `<`, `contains`, `is_empty`, `is_not_empty`

### `logic_operator`

`"AND"` (all conditions must be true) or `"OR"` (any condition must be true). Default: `"AND"`.

### `on_true` / `on_false` Actions

| Action | Meaning |
|---|---|
| `continue` | Proceed to next step |
| `warn` | Log a yellow warning banner and continue |
| `stop` | Stop the flow without error |
| `raise_error` | Stop the flow and mark it as failed |
| `run_flow` | Run a sub-flow — set `target` to the flow name |
| `jump` | Jump to a named step — set `target` to the step name |

For `run_flow`, optionally pass `subflow_inputs`:
```json
{
  "action": "run_flow",
  "target": "cleanup_flow",
  "subflow_inputs": { "user_id": "{{context.step_1.userId}}" }
}
```

### Python Code Mode — Available Helpers

The snippet runs with the full helper environment from `build_exec_globals` (`code_generator/generate_test_suite/helpers/flow_context.py`). Return one of the **flow-control** helpers to decide what happens next:

| Return value | Effect | Maps to action |
|---|---|---|
| `continue_flow()` | proceed to next step | `continue` |
| `stop_flow()` | stop the flow without error | `stop` |
| `raise_error(message="...")` | stop and mark the flow failed | `raise_error` |
| `warn_flow(message="...")` | log a warning and continue | `warn` |
| `run_subflow(flow_name, inputs={})` | run a sub-flow | `run_flow` |
| `jump_to_step(step_name)` | jump to a named step | `jump` |

**Utility helpers** available inside the snippet (do not need to be returned):

- `get_var("step_N.fieldName", default=None)` — read any context value (always use full `step.field` format)
- `set_var(name, value)` — export a value for later steps
- `get_env(name, default=None)` — read an environment variable
- `parse_json(text)` / `to_json(obj, pretty=False)` — JSON helpers
- `get_response(path, default=None)` — read from the last response
- `mask(value, visible=4)` — mask a secret for logging
- `assert_that(expr, message)` / `expect(actual, op, expected, msg)` — inline assertions
- `now(fmt)`, `uuid()`, `sleep(seconds)`, `retry(fn, times=3, delay=1.0)`
- `log(message, level)` — log a message without stopping; `dump_context()` — print all context keys
- Any **user helpers** under `user_data/user_helpers/*.py` (flat names and module-namespaced)

---

## `flow` — Run a Sub-Flow as a Step

Invokes another complete flow as a single step within the current flow.

### Schema

```json
{
  "type": "flow",
  "name": "step_3",
  "flow_name": "member_auth_flow",
  "context_export": ["accessToken"],
  "context_import": [],
  "flow_inputs": {
    "feature_id": "{{context.flow_input.feature_id}}"
  }
}
```

### Field Reference

| Field | Required | Notes |
|---|---|---|
| `flow_name` | **yes** | Name of an existing flow — must already exist before this flow is built |
| `context_export` | no | Fields exported from the sub-flow into the parent's context |
| `context_import` | no | Values to bring in from parent context before running sub-flow |
| `flow_inputs` | no | Map of input values to pass into the sub-flow's declared inputs |

### Sub-flow Result Merging

After the sub-flow completes, its context is merged into the parent context. These flow-level keys are **never overwritten**: `flow_name`, `steps_executed`, `all_steps_passed`, `errors`, `expected_flow_execution_time`.

---

## `wait` — Simple Time Delay

A step that only pauses — no API call or query is made.

```json
{
  "type": "wait",
  "name": "pause_before_poll",
  "waitBefore": 3
}
```

Prefer `waitBefore`/`waitAfter` on other step types over a standalone `wait` step. Use `wait` only when the sole purpose is the delay itself.
