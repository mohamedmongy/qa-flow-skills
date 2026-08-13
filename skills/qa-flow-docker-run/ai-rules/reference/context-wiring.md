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

---

## Injecting Context Into Payload / Params — Template Variables

Use the full `{{context.<path>}}` syntax inside `payload`, `params`, and `query_params`:

| Source | Template syntax |
|---|---|
| Previous step response | `{{context.step_name.fieldName}}` |
| Flow input | `{{context.flow_input.inputName}}` |
| Environment variable | `{{context.env.VAR_NAME}}` |
| session_config header | `{{context.session_config.headerName}}` |

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

For environment variables specifically, `context_substitution.py` resolves three forms (in addition to `{{context.env.VAR}}`):

| Form | Behavior |
|---|---|
| `{{env.VAR_NAME}}` | Reads `VAR_NAME` from the active environment (`os.environ`) |
| `{{env.VAR_NAME\|default}}` | Same, but falls back to `default` when the variable is unset |
| `{{env.<slug>.VAR_NAME}}` | Always reads from the named `.env.<slug>` file (e.g. `{{env.live.BASE_URL}}` reads `.env.live`) regardless of the active environment |

`VAR_NAME` must be uppercase (`[A-Z_][A-Z0-9_]*`); the slug must be lowercase.

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
