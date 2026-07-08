# QA Flow — Step Type: `api_call`

The most common step type. Executes one or more test cases from an existing API definition.

## Full Schema

```json
{
  "type": "api_call",
  "name": "get_csrf_token",
  "api": "get_csrf_token",
  "test_case": "default",
  "payload": {},
  "params": {},
  "context_export": ["csrfToken"],
  "context_import": ["previous_step.someField"],
  "header_import": [],
  "waitBefore": 0,
  "waitAfter": 0,
  "file_downloads_code": ""
}
```

## Field Reference

| Field | Required | Notes |
|---|---|---|
| `api` | **yes** | Must match an existing API definition name exactly |
| `test_case` | yes (or `test_cases`) | Single test case name string |
| `test_cases` | yes (or `test_case`) | Array of test case names — runs all sequentially |
| `payload` | no | Overrides or supplements the test case's default payload. Step payload takes precedence. |
| `params` | no | Query string parameter overrides. Same merge rule as `payload`. |
| `context_export` | no | **List** of response field names to save into flow context (canonical field) |
| `response_export` | no | Legacy alias for `context_export` — prefer `context_export`; don't set both |
| `context_import` | no | Explicit dependency declaration (validates existence, does not inject automatically) |
| `header_import` | no | Injects context values as request headers |
| `waitBefore` | no | Seconds to wait before executing (default 0) |
| `waitAfter` | no | Seconds to wait after executing (default 0) |
| `file_downloads_code` | no | Python snippet to save response files to Assets Manager |

## Multiple Test Cases — "Last Wins"

When `test_cases` is an array, all test cases run sequentially. Each one's response overwrites the previous export. Only the **last** test case's response is exported to context. Step is marked **failed** if any test case fails.

```json
{ "test_cases": ["valid_login", "invalid_tenant", "invalid_csrf_token"] }
```

## Payload Merge Order

1. Test case's own `payload` is loaded as the base
2. Step-level `payload` fields are merged on top (step wins on conflicts)
3. `{{context.*}}` placeholders in the merged payload are resolved against live flow context
4. If `payload` is defined on the step, its fields are auto-exported to context

## Critical Rules

- There is **no** `assertions` field on a step — assertions live inside the API's test case definition
- There is **no** `headers` field on a step — use `header_import` for dynamic headers
- `api` must reference an existing API definition — call `list_api_definitions` to verify
- `test_case` must match an existing test case exactly — call `get_test_cases` to verify (a wrong name silently passes without testing anything)

## Example: Simple API Call with Context Wiring

```json
{
  "type": "api_call",
  "name": "login",
  "api": "login_with_otp",
  "test_case": "valid_login",
  "payload": {
    "phone": "{{context.flow_input.user_phone}}"
  },
  "context_import": ["get_csrf.csrfToken"],
  "header_import": [
    { "header": "X-CSRF-Token", "prefix": "", "variable": "get_csrf.csrfToken" }
  ],
  "context_export": ["accessToken", "userId"]
}
```

## Example: Step-Level Payload Override with Auto-Generated Values

```json
{
  "type": "api_call",
  "name": "create_user",
  "api": "create_user_api",
  "test_case": "default",
  "payload": {
    "email": "{{auto.email}}",
    "referral_code": "{{context.step_1.promoCode}}"
  }
}
```

Auto-generated value options: `{{auto.email}}`, `{{auto.phone}}`, `{{auto.uuid}}`, `{{auto.timestamp}}`, `{{auto.string}}`, `{{auto.number}}`
