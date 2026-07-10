# QA Flow API Negotiation Rule

## Scope
These rules apply to **any AI assistant** whenever a user requests the creation or update of an **API definition** using the QA Flow MCP tools (`create_or_update_api`).

---

## Core Principle — Negotiate Before You Build

**Never call `create_or_update_api` as the first response to a user request.**

**No tool calls before negotiation begins.** The FIRST response to an API creation request must be the Step 2 name question — **not** `list_api_definitions`, `get_api_definition`, `health_check`, or any other read-only tool. "Gathering context first" is the forbidden pattern itself. Context-gathering happens in Step 1, **after** the name question is answered.

Always go through the negotiation steps below in order.

---

## Step 1 — Gather Context from the MCP Server (After the Name Is Confirmed)

Only **after** the user has answered the Step 2 name question, call `list_api_definitions` to understand what already exists.

- Is there an existing API hitting the same endpoint?
- What naming patterns are in use (e.g. `get_csrf_token`, `csrf_v2`, `csrf_mcp_v2`)?
- If a similar API exists, call `get_api_definition` on it to inspect its headers, test cases, and structure.

Use this context to inform the remaining questions — never to skip them. (This mirrors the flow rule's "zero tool calls before 2a" gating in `ai-rules/negotiation/flow.md`.)

### Name Collision Check (run before asking question 2)

The name confirmed in question 1 must be checked against the `list_api_definitions` result **before** moving on:

- **Exact match exists** → do **not** silently proceed: `create_or_update_api` with an existing name **updates/overwrites** that API. Tell the user it already exists (show its method + endpoint), and ask one question offering **2–3 alternative names** derived from the *Naming Conventions* below — a versioned suffix (`csrf` → `csrf_v2`), a more specific action or resource word (`login` → `login_with_otp`), or a service/scope prefix matching sibling APIs — plus the explicit option to **update the existing API instead**. Example:

  ```
  An API named `get_csrf_token` already exists (GET {{env.BASE_URL}}/.../auth/csrf). Should I
  (1) name this one `get_csrf_token_v2`  (2) `get_member_csrf_token`  (3) update the existing `get_csrf_token`  (4) type another name
  ```

- **Near-collision** — a differently named API already hits the **same method + endpoint** → surface it the same way and ask whether to keep the chosen name (a true variant), reuse the existing API as-is, or update it.

Only when the name is resolved (unique, or the user explicitly chose to update the existing one) continue to question 2.

**Turn the context into pickable options, not prose.** When a similar API exists, seed the remaining questions with its values as concrete options the user can pick with one tap/keystroke instead of typing — e.g. offer its endpoint as *"same as `csrf`"* vs *"different (paste cURL)"*, mirror its test-case set as a suggested scenario list, and propose its tags/priority as the recommended choice. Put the recommended option first. This never replaces asking a question; it only makes answering it faster.

---

## Step 2 — Ask One Question at a Time (Sequential, Strictly)

Based on the user's cURL and the server context, identify only the gaps that are actually ambiguous — then ask about them **one at a time**, in strict order. 

**Critical rules:**
- **Never bundle multiple questions into a single message — ever.**
- **Never skip a question because a similar API exists or because you think you already have enough context.** A similar API is reference material only, not a substitute for the user's input.
- **Never present a full summary or proposed plan until all four questions have been answered.** Do not jump to Step 3 early.
- **After each answer, ask only the next question — nothing else.**
- **If the user's answer only addresses one question (e.g. just gives a name), do not infer the rest. Ask the next question.**

Ask in this exact order, one message per question:

1. **Name** — What should this API be called? **When a cURL is provided, don't ask open-ended — detect a proposed name from it first:** derive a `snake_case` name from the HTTP method and the meaningful trailing path segment(s) of the endpoint, following the *Naming Conventions* below (e.g. `GET .../auth/csrf` → `get_csrf_token`, `POST .../blacklist/bulk-upload` → `blacklist_bulk_upload`). Present it as the recommended option the user can accept with one tap, alongside a free-text alternative — e.g. `(1) get_csrf_token (recommended)  (2) type your own`. Wait for the answer before continuing. The duplicate check against the server happens immediately after, in Step 1 (see *Name Collision Check*) — never claim the name is available before that check has run.
2. **Test cases** — Ask in plain English: *"What scenarios should this API cover? e.g. successful export, invalid GUID, missing token — describe them however you like."* Wait for the answer. Derive status codes and overrides from the answer. Do not invent scenarios.
3. **Assertions** — *"Do you have specific assertions to validate in the response (e.g. a body field, a message)?"* Wait for the answer. Do not skip this and do not add assertions speculatively.
4. **Tags and priority** — *"Should test cases be tagged (e.g. smoke, positive, negative) and given a priority (high, medium, low)?"* Wait for the answer.

Do not ask about things the user already provided in the cURL or that the server already answered.

Several Step 2 follow-ups are **small fixed choice sets** — the expected status of a negative case (400/401/403/…), tags, priority, and each static-value placement (env var vs literal). Present those as option menus per the selection format rule below (native picker where the client has one), seeding the options from the reference API's values when Step 1 found one. Only genuinely open questions (name, scenario descriptions, body assertions) stay free-text.

> **Selection format rule:** small fixed choice sets → an inline numbered menu on one line — `(1) a  (2) b  (3) c`. Long lists of existing items → a numbered table/list, one option per line with a short identifying detail; show at most **20** (tell the user more exist and to just name theirs); prefix rows with `- [ ]` when several can be picked. Full rule, examples, and native structured-UI guidance: read `ai-rules/reference/selection-format.md` before presenting your first list.

### ⚠️ Static Values → Env Vars (or Flow Inputs)

While deriving payload/params and headers from the cURL or the user's answers, **do not silently hardcode environment-specific, shared-config, or secret literals** (base URLs, host names, client IDs, tenant IDs, status IDs, API keys, Bearer tokens). For each such value, ask — one question, on its own — whether it should be replaced with an env var placeholder:

```
The cURL has `tenant: montymobile`. Should I (1) replace it with an env var `{{env.TENANT}}`, or (2) keep it literal in the test case?
```

- **Environment-specific / shared / secret** → recommend an **env var**: use `{{env.VAR_NAME}}` in `payload`/`params` and `headers`. If the var doesn't exist yet, tell the user it must be added to the environment config and name the variable you propose. Base URLs **always** use `{{env.BASE_URL}}` (see *Environment Variable Placeholders*); never hardcode a host.
- **Run-varying test data** (IDs, GUIDs, phone numbers) that a flow will supply → note it should come from a flow input when this API is used as a flow step (`{{context.flow_input.X}}`), not be hardcoded in the test case.
- **Genuinely constant for every run and environment** → only then keep it literal, after the user confirms.

Default recommendation: **env var** for config/secrets, never assume a hardcoded host or token.

---

## Step 3 — Present a Proposed Plan and Ask for Confirmation

Only after **all four questions above have been answered**, present the full summary:

- API name, method, and endpoint URL
- Each test case: identifier (`test_case`), description, expected status, assertions, payload/params overrides, tags, priority
- **Static values promoted** — for each config/secret literal from the cURL, state whether it became an env var (and which one, noting any that must still be created) or was intentionally kept literal

Then ask: **"Does this look right, or would you like to change anything before I build it?"**

Do not proceed until the user explicitly confirms. If the user requests any change (e.g. a different name), apply only that change and re-present the summary for confirmation. Do not build until confirmed.

---

## Step 4 — Build

Only after explicit user confirmation, call `create_or_update_api` with the agreed parameters.

Follow the rules in `ai-rules/safeguard.md` during this step.

---

## Established Patterns — Apply These When Building

These patterns are derived from the existing APIs in this project. Match them when creating new ones.

### Naming Conventions
- Use `snake_case` for all API names: `get_csrf_token`, `login_with_otp`, `export_campaign_csv`
- Versioning suffix when a variant exists: `csrf_v2`, `csrf_mcp_v2`, `blacklist_bulk_upload_v2`
- Descriptive names that reflect the action: `get_`, `login_`, `export_`, `upload_`

### Environment Variable Placeholders
- Base URL: always use `{{env.BASE_URL}}` instead of hardcoded hostnames when the API belongs to the main application
  - Example: `{{env.BASE_URL}}/api-gateway/member/api/v1/auth/csrf`
- Secondary services have their own env vars: `{{env.HTTP_BIN}}` for httpbin-style endpoints
- Asset references use `{{assets.<key>}}`: e.g. `{{assets.blacklistkey}}` for file uploads
- Never hardcode Bearer tokens in the curl — if an Authorization header is present, flag it and ask whether it should be replaced with `{{env.TOKEN}}` or a flow-extracted value

### Test Case Structure
Each test case supports the following fields — use only what's relevant:

The authoritative schema is the `qa-flow://schemas/test-case` MCP resource (`mcp_server/schemas.py`). The canonical fields are:

```json
{
  "test_case": "snake_case_identifier",
  "description": "what this case tests",
  "expected_status": 200,
  "expected_execution_time": 3.5,
  "payload": {},
  "params": {},
  "assertions": [],
  "context_export": {},
  "tags": [],
  "priority": "high"
}
```

- `test_case`: required, snake_case, unique within the API (e.g. `valid_csrf_token`, `invalid_tenant`, `valid_blacklist_bulk_upload`) — this **is** the identifier; there is no separate `name` field on a test case
- `description`: brief description of what the case validates
- `expected_status`: required HTTP status code integer (default `200` if omitted)
- `expected_execution_time`: default `3.5` — max seconds before the test is flagged slow
- `payload`: request body fields to override for this test case (empty `{}` if none); values support `{{env.VAR_NAME}}` and `{{auto.*}}`
- `params`: query string parameters to override for this test case
- `assertions`: list of response assertions (see below)
- `context_export`: response values exported for later flow steps. **At the test-case level this is a dict** (`{"var_name": "response.path.to.value"}`), unlike the flow-step `context_export`, which is a list
- `tags`: optional list of pytest markers — common ones: `smoke`, `regression`, `integration`, `negative`
- `priority`: optional, values: `critical`, `high`, `medium`, `low`

> ⚠️ **No per-test-case `headers` field.** The generator does **not** read a `headers` key on a test case — request headers come from the global `headers` map in `user_data/global_data.py`, merged with any flow-level `header_import` (and `session_config` when shared session is on). To vary a header per scenario (e.g. an `invalid_tenant` case), inject it through the flow step's `header_import`, not a test-case `headers` field. This mirrors the "no `headers` field on a step" rule in `ai-rules/reference/build-workflow.md`.

### Assertions
Assertions validate response fields beyond the status code. Format:

```json
{
  "field": "status_code",
  "operator": "==",
  "value": 200
}
```

Supported operators: `==`, `!=`, `contains` (any other operator silently evaluates to a failed assertion — these three are the only ones the generated assertion check handles).

Common assertion `field` values (resolved by the generated test code):
- `status_code` — the HTTP status code
- `response.<field_path>` — a field in the JSON response body. The field **must** start with `response.` (e.g. `response.data.message`); the generator strips the `response.` prefix and walks the parsed body. A bare `body.<...>` path is **not** recognized.
- Any other bare name is looked up in the step/test context, not the response body

When to suggest assertions:
- Always include a `status_code` assertion when using the `assertions` array format
- Suggest body assertions when the user mentions validating a specific response field or message (e.g. `expected_message`) — express them as `response.<path>`
- Do not add assertions speculatively — only include them if the user has indicated what to validate

### Test Case Naming Conventions
Follow the patterns seen across existing APIs:
- Happy path: `default`, `valid_<action>`, e.g. `valid_csrf_token`, `valid_login`, `valid_blacklist_bulk_upload`
- Negative cases: `invalid_<what_is_wrong>`, e.g. `invalid_tenant`, `invalid_csrf_token`, `invalid_whatsapp_campaign_missing_phone`
- Specific scenarios: descriptive snake_case, e.g. `valid_whatsapp_campaign_contact`, `get_all_valid`

### Content Type
Set the API-level `content_type` to match the request (it is read from the API definition, not per test case):
- `application/json` — default for JSON APIs
- `multipart/form-data` — for file upload endpoints (e.g. bulk upload)
- `application/x-www-form-urlencoded`, `raw`, or `binary` — for the corresponding body styles (with `raw_content_type` when using `raw`/`binary`)

---

## What This Rule Is Not

- It is not a rigid questionnaire. Questions are derived from the specific request and server context — not a fixed checklist.
- It does not block quick edits where intent is already clear and confirmed.
- It does not apply when the user is asking for information only (e.g. "show me the API", "list all APIs").
