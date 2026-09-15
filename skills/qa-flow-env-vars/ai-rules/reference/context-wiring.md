# QA Flow — Context Wiring (Import, Export, Headers, Templates)

Context is the mechanism for passing data between steps in a flow. The generator enforces strict step-prefixed key format — using bare field names causes runtime validation errors.

---

## Exporting Values FROM a Step

Use **`context_export`** — it is the canonical field for saving named response fields into the shared flow context. `response_export` is a legacy alias that resolves to the same behavior; prefer `context_export` in all new flows and **do not set both on the same step**.

```json
{
  "context_export": ["csrfToken", "accessToken"]
}
```

The value is stored under the key `<step_name>.<fieldName>`:

| Step name | Exported field | Context key |
|---|---|---|
| `get_csrf` | `csrfToken` | `get_csrf.csrfToken` |
| `login` | `accessToken` | `login.accessToken` |
| `step_4` (query) | `result.bundle_id` | `step_4.result.bundle_id` |

**Always a LIST** — never a dict or a bare string.

### Path Resolution in context_export

- Simple field `"csrfToken"` → looks up `csrfToken` directly in response body
- Dot path `"data.csrfToken"` → resolves `response.data.csrfToken`
- The `"response."` prefix is stripped automatically before lookup
- Array indexing `"result[0].guid"` → resolves the first element's `guid`

### Steps With Multiple Test Cases — the LAST one wins

A step that selects several test cases (`test_cases`, on `api_call` or `wait_until`) exports once per test case, each overwriting the last. Only the **final** test case's response survives under `<step_name>.<field>` — so the order of `test_cases` decides what every later step reads.

```json
{ "name": "login", "test_cases": ["invalid_password", "valid_login"], "context_export": ["accessToken"] }
// login.accessToken = the accessToken from valid_login (listed last)
```

Listing a negative case last exports its error body — or `None` — and silently breaks the downstream `context_import` / `header_import` / `{{context.*}}` that expected a real value. Order the array so the case that produces the value comes last; see `api-and-testcase-selection.md`.

---

## Declaring Dependencies with context_import

`context_import` is a declarative dependency list. It validates existence before the step runs — it does **not** inject values automatically.

```json
{
  "context_import": ["get_csrf.csrfToken", "step_2.accessToken"]
}
```

- Always use the full `<step_name>.<field>` format
- Never reference a future step — the context manager validates execution order and will error

---

## Injecting Context Into Request Headers — header_import

`header_import` injects a context value directly as an HTTP request header on the step.

```json
"header_import": [
  {
    "header": "X-CSRF-Token",
    "prefix": "",
    "variable": "get_csrf.csrfToken"
  },
  {
    "header": "Authorization",
    "prefix": "Bearer ",
    "variable": "step_2.accessToken"
  },
  {
    "header": "RecaptchaToken",
    "prefix": "",
    "variable": "session_config.RecaptchaToken"
  },
  {
    "header": "X-Feature-Id",
    "prefix": "",
    "variable": "flow_input.feature_id"
  }
]
```

| Field | Required | Notes |
|---|---|---|
| `header` | yes | Exact HTTP header name |
| `prefix` | yes | Prepended to the resolved value. Use `"Bearer "` for auth tokens, `""` for raw values |
| `variable` | yes | Context path in **short form** — no `context.` prefix |

### Variable Source Formats for header_import.variable

| Source | Short form (header_import) |
|---|---|
| Previous step response | `<step_name>.<fieldName>` |
| session_config header | `session_config.<headerName>` |
| Environment variable | `env.<VAR_NAME>` |
| Flow input | `flow_input.<inputName>` |
| Dataset row column (data-driven flow) | `data.<column>` |

---

## Injecting Context Into Payload / Params — Template Variables

Use the full `{{context.<path>}}` syntax inside `payload`, `params`, and `query_params`:

| Source | Template syntax |
|---|---|
| Previous step response | `{{context.step_name.fieldName}}` |
| Flow input | `{{context.flow_input.inputName}}` |
| Environment variable | `{{context.env.VAR_NAME}}` |
| session_config header | `{{context.session_config.headerName}}` |
| Dataset row column (data-driven flow) | `{{data.column}}` (or `{{context.data.column}}`) |

```json
{
  "payload": {
    "phone":       "{{context.flow_input.user_phone}}",
    "client_id":   "{{context.env.CLIENTID}}",
    "promo_code":  "{{context.step_1.promoCode}}"
  }
}
```

**Never** use `{{step_1.field}}` without the `context.` prefix — it will not be resolved.

### Environment Variable Template Forms

The full syntax for environment variables — `{{context.env.VAR}}`, `{{env.VAR}}`, `{{env.VAR|default}}`, the pinned `{{env.<slug>.VAR}}`, `env.VAR` in `header_import`, `get_env()` in `python_code` — and which form belongs in which location is defined once in **`ai-rules/negotiation/env-vars.md`** §2. Read it before writing any env reference: a form used in the wrong location is sent as literal text.

### Dataset Row Values (Data-Driven Flows)

When the flow carries a `dataset` binding, the current row is exported under the `data` prefix before any step runs: `{{data.<column>}}` in payload/params, `data.<column>` in `header_import` / `context_import`, `get_var('data.<column>')` in Python-code conditions. A string that is exactly `{{data.col}}` keeps the column's native type. See `ai-rules/reference/dataset-binding.md`.

### Auto-Generated Values (No Context Needed)

| Template | Produces |
|---|---|
| `{{auto.email}}` | Unique email address |
| `{{auto.phone}}` | Unique phone number |
| `{{auto.uuid}}` | UUID v4 |
| `{{auto.timestamp}}` | Current Unix timestamp |
| `{{auto.string}}` | Random string |
| `{{auto.number}}` | Random number |

---

## Context Key Format Rules

| Rule | Correct | Wrong |
|---|---|---|
| `context_import` entries | `"step_1.csrfToken"` | `"csrfToken"` |
| `header_import.variable` | `"step_1.accessToken"` | `"context.step_1.accessToken"` |
| Payload template variables | `{{context.step_1.token}}` | `{{step_1.token}}` |
| `context_export` shape | `["csrfToken"]` (list) | `{"csrfToken": "path"}` (dict) |
| Execution order | Reference only earlier steps | Referencing a future step → runtime validation error |
| DB query result fields | `"result.status"` | `"status"` |

---

## Complete Data-Flow Example

```json
[
  {
    "name": "get_csrf",
    "type": "api_call",
    "api": "get_csrf_token",
    "test_case": "default",
    "context_export": ["csrfToken"]
  },
  {
    "name": "login",
    "type": "api_call",
    "api": "login_with_otp",
    "test_case": "valid_login",
    "context_import": ["get_csrf.csrfToken"],
    "header_import": [
      { "header": "X-CSRF-Token",    "prefix": "",        "variable": "get_csrf.csrfToken" },
      { "header": "RecaptchaToken",   "prefix": "",        "variable": "session_config.RecaptchaToken" }
    ],
    "context_export": ["accessToken", "userId"]
  },
  {
    "name": "fetch_profile",
    "type": "api_call",
    "api": "get_user_profile",
    "test_case": "default",
    "params": { "user_id": "{{context.login.userId}}" },
    "context_import": ["login.accessToken"],
    "header_import": [
      { "header": "Authorization", "prefix": "Bearer ", "variable": "login.accessToken" }
    ]
  }
]
```
