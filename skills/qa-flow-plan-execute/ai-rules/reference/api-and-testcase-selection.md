# QA Flow — API Selection and Test Case Selection

Before wiring any step, verify which APIs and test cases actually exist in the system. Using a name that doesn't exist causes the step to silently pass without testing anything.

---

## Step 1 — Discover Available APIs

```
list_api_definitions
```

- Only APIs that exist in the system can be referenced in a step's `api` field
- If the required API doesn't exist, create it first with `create_or_update_api` (follow `ai-rules/negotiation/api.md`)
- The `api` field in a step must exactly match the API definition's `name` — case-sensitive

---

## Step 2 — Discover Available Test Cases

```
get_test_cases(api_name: "my_api")
```

Always call `get_test_cases` before wiring any step to:

- See exactly which test case names exist (e.g., `default`, `valid_login`, `invalid_tenant`)
- See each test case's `payload`, `params`, `expected_status`, and `assertions`
- Avoid typos — a non-existent `test_case` name silently runs without testing anything

---

## Wiring a Single Test Case

```json
{ "test_case": "valid_login" }
```

The name must match exactly what `get_test_cases` returns.

---

## Wiring Multiple Test Cases in One Step

```json
{ "test_cases": ["invalid_tenant", "invalid_csrf_token", "valid_login"] }
```

- All test cases run sequentially on the same API instance, **in array order**
- Each test case starts from **its own payload/params** from the API definition, but any key the step sets is sent to **every** selected test case (see the merge rule below) — so leave a key out of the step when test cases need different values for it
- Context export uses the **last** test case's response ("last wins")
- Step is marked **failed** if any single test case fails; the remaining test cases still run

`test_cases` must be an **array** — passing a string causes a format error.

The same applies to a `wait_until` step (see `step-wait-until.md`): every selected test case is polled in turn, each with its own full timeout window.

### ⚠️ ORDER MATTERS — the last test case feeds the context

The array order is not cosmetic. It is both the execution order and the export contract:

| What depends on order | Rule |
|---|---|
| `context_export` / `response_export` | Every test case overwrites the previous export; only the **last** one's values survive into `<step>.<field>` |
| Later steps (`context_import`, `header_import`, `{{context.*}}`) | They read whatever the **last** test case produced |
| Step status | Independent of order — any failure fails the step |

**Therefore: put the test case whose response later steps consume LAST.** A negative case placed last exports its error body (or `None`) and breaks every downstream step that expected a token/id.

```json
// ✅ correct — the happy path is last, so login.accessToken is a real token
{ "name": "login", "test_cases": ["invalid_password", "valid_login"], "context_export": ["accessToken"] }

// ❌ wrong — accessToken is exported from the failing negative case
{ "name": "login", "test_cases": ["valid_login", "invalid_password"], "context_export": ["accessToken"] }
```

If a step both exports context and needs negative cases in a specific reporting order, split it into two steps instead of relying on export order.

---

## Overriding a Test Case's Payload at the Step Level

Provide a `payload` on the step to override or supplement the test case's default payload:

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

**Merge rule (one or many test cases, `api_call` and `wait_until` alike):** step-level `payload` keys take precedence over the test case's own payload keys — a step value referencing `{{data.*}}` becomes the dataset row's value, a literal is sent as-is — and the test case fills only the keys the step does not set. Both are merged, then every placeholder (`{{data.*}}`, `{{context.*}}`, `{{env.*}}`, `{{auto.*}}`) is resolved, whether it came from the step or the test case. Overridden keys are logged at runtime (`🔀 Step payload overrides test case '…' value(s) for: …`).

With several test cases selected, a step key therefore reaches **all** of them: a negative test case that relies on its own bad value must not have that key set on the step. The dashboard pre-fills the step's Payload box from the test case picked in *Single* mode — remove those copied keys before switching a step to several test cases.

Same rules apply to `params` for GET/DELETE query string parameters.

---

## Checklist Before Wiring Any Step

1. `list_api_definitions` → confirm the API exists
2. `get_test_cases(api_name)` → note the exact test case name(s) to use
3. Inspect each test case's payload/params → decide if step-level overrides are needed
4. Confirm what the API response looks like → plan which fields to add to `context_export`
5. If more than one test case is selected → confirm the **order**, with the case that feeds `context_export` last

---

## Common Naming Patterns (from existing APIs in this project)

| Pattern | Examples |
|---|---|
| Happy path | `default`, `valid_csrf_token`, `valid_login`, `valid_blacklist_bulk_upload` |
| Negative cases | `invalid_tenant`, `invalid_csrf_token`, `invalid_whatsapp_campaign_missing_phone` |
| Specific scenarios | `valid_whatsapp_campaign_contact`, `get_all_valid` |
