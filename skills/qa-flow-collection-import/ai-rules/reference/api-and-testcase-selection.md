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
{ "test_cases": ["valid_login", "invalid_tenant", "invalid_csrf_token"] }
```

- All test cases run sequentially on the same API instance
- Context export uses the **last** test case's response ("last wins")
- Step is marked **failed** if any single test case fails

`test_cases` must be an **array** — passing a string causes a format error.

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

**Merge rule:** step-level `payload` keys take precedence over the test case's own payload keys. Both are merged, then `{{context.*}}` placeholders are resolved.

Same rule applies to `params` for GET/DELETE query string parameters.

---

## Checklist Before Wiring Any Step

1. `list_api_definitions` → confirm the API exists
2. `get_test_cases(api_name)` → note the exact test case name(s) to use
3. Inspect each test case's payload/params → decide if step-level overrides are needed
4. Confirm what the API response looks like → plan which fields to add to `context_export`

---

## Common Naming Patterns (from existing APIs in this project)

| Pattern | Examples |
|---|---|
| Happy path | `default`, `valid_csrf_token`, `valid_login`, `valid_blacklist_bulk_upload` |
| Negative cases | `invalid_tenant`, `invalid_csrf_token`, `invalid_whatsapp_campaign_missing_phone` |
| Specific scenarios | `valid_whatsapp_campaign_contact`, `get_all_valid` |
