# QA Flow Collection Import Negotiation Rule

## Scope
These rules apply to **any AI assistant** whenever a user requests the **bulk import of a request collection** — a **Postman** collection (schema `.../collection/v2.x.x`) or a **Swagger / OpenAPI** spec (`swagger: 2.0` / `openapi: 3.x`) — into QA Flow **API definitions** using the `create_or_update_api` MCP tool.

This rule governs the **bulk workflow**. It does **not** replace `negotiation/api.md` — it **composes with it**: every API this workflow produces must still conform to `api.md` (naming, test-case structure, the *Static Values → Env Vars* rule, content type). Read `api.md` alongside this file. All env-var creation and the per-batch confirmation obey `safeguard.md`.

Use this rule when the source is a whole collection/spec (many requests). For a **single** cURL or one endpoint, use `negotiation/api.md` directly instead.

---

## Core Principle — Negotiate the Conventions Once, Build in Confirmed Batches

**Never call `create_or_update_api` before both (a) the global conventions and (b) the current folder's batch plan have been confirmed by the user.**

A collection can hold hundreds of requests across dozens of folders. Negotiating each request individually per `api.md` is impractical, and building them silently is forbidden. Instead:

1. **Parse the collection** (a local file read — allowed; this is not an MCP call) and show the user an inventory.
2. **Negotiate the *global* conventions once** — variable/URL mapping, auth, naming, test-case policy, batching — one topic per message.
3. **Confirm a per-folder batch plan**, then build that folder's APIs. Repeat per folder.

The **only** MCP tool permitted before a batch is built is the single `list_api_definitions` name-availability check for that batch (Step 4). No other MCP tool runs before negotiation — not `health_check`, not `get_api_definition`, not `manage_environment_variables` (env vars are created only after their mapping is confirmed, inside Step 5). "Gathering context first" beyond reading the file and that one listing is the forbidden pattern itself.

Always go through the steps below in order.

---

## Step 1 — Get the Source (Only If Missing)

If the request does not already point to a collection/spec (a file path, an attached file, or pasted JSON/YAML), the first response is — with **zero tool calls**:

> *"Point me at the collection to import — a Postman collection (`.postman_collection.json`) or a Swagger/OpenAPI spec (path or pasted). Which one?"*

**Skip this entirely when the source is already provided** — never re-ask for what's in the request.

---

## Step 2 — Parse and Present the Inventory

Read the file and present a **read-only** summary in one message so the user sees the scope before any decision:

- **Format & version** — Postman v2.1 collection, or OpenAPI 3.x / Swagger 2.0.
- **Folders and request counts** — a table of top-level folders with the number of requests in each, and the total. Cap the list per `reference/selection-format.md` (show ≤20, note the rest).
- **Detected variables** — every `{{var}}` used across the collection (Postman) or every `servers[]` / `securityScheme` / shared parameter (OpenAPI). Flag obvious typo-duplicates (e.g. `tenant` vs `teenant`, `token` vs `tokeen`, `member-url` vs `MemberBaseUrl`) so the user can decide whether they collapse to one env var.

This is a summary, not a question. Follow it immediately with Step 3's first question.

---

## Step 3 — Negotiate the Global Conventions (One Topic per Message)

Ask these **one message at a time**, in order. Related low-priority settings are grouped into a single preset-menu question per `reference/selection-format.md`; never bundle unrelated topics. Wait for each answer. These answers become the **convention ledger** applied to every API in every batch — restate the ledger if the conversation is long enough that it scrolls out of view (`safeguard.md` Rule 7).

1. **Variable / URL mapping.** Present the detected variables as a table and, for each, ask how it resolves — an env var placeholder, a literal, or a run-varying flow input. Follow `api.md`'s *Static Values → Env Vars* rule: base URLs **always** become `{{env.BASE_URL}}` (never a hardcoded host); tenants/client IDs/versions become env vars or confirmed literals; secrets become env vars.
   - Example resolutions (confirm, don't assume): `{{member-url}}` → `{{env.BASE_URL}}/api-gateway/member`; `{{version}}` → literal `v1`; `{{tenant}}` (+ any typo variant) → `{{env.TENANT}}`.
   - **Flag every env var that does not yet exist** in the target environment — it must be created via `manage_environment_variables` in Step 5 **before** the APIs that reference it, or the dashboard rejects the `create_or_update_api` call.

2. **Authentication.** How should the request auth (Postman `auth: bearer {{token}}`, or an OpenAPI `securityScheme`) be handled — `{{env.TOKEN}}` (recommended, mark **sensitive** with a replace-me placeholder), a value a login flow extracts later, or a confirmed literal? One question.

3. **Naming convention.** Confirm how each request name maps to an API `snake_case` name (default: `action_resource`, e.g. *Get All Policies* → `get_all_policies`, *Assign Client Countries* → `assign_client_countries`), and how folder context participates when request names collide across folders (prefix vs suffix). Collisions with **existing** APIs are resolved per Step 4, never by overwrite.

4. **Test-case policy** (grouped — one message). For the auto-generated test case on each API: happy-path only (`valid_<api_name>`) vs. also negative cases; default `expected_status` (200); default assertion (`status_code == 200`, plus any body assertion policy); default `tags` (e.g. `smoke, positive`) and `priority`. Sample GUIDs/IDs in request bodies stay as default test-case params unless the user says otherwise (flows override them later via flow input).

5. **Batching.** Which folders to import and in what order, built **one folder per batch** (each folder confirmed on its own in Step 4). Confirm the starting folder.

Do not present a proposed plan or call any tool until all five are answered.

---

## Step 4 — Per-Batch Plan and Confirmation

For the current folder (batch), and **only now**:

1. Call `list_api_definitions` **once** — the single name-availability check for this batch. Apply the naming convention to every request, then compare against existing names. On a collision, resolve to a **new name** (versioned/scoped alternative from `api.md`'s conventions) — **never overwrite** an existing API. Also flag near-collisions (same method + endpoint under a different name) so the user can reuse instead of duplicating.
2. Present the batch as a table: **request → `api_name`, method, resolved endpoint, promoted variables, content type, test case(s)**. List the env vars that will be created first and which are still placeholders.
3. Ask: **"Build these N APIs from folder `<name>`, or change anything first?"** Do not build until the user confirms this batch.

---

## Step 5 — Build the Batch

Only after the batch is confirmed:

1. **Create any missing env vars first** via `manage_environment_variables` (follow `safeguard.md`) — the dashboard rejects a curl whose `{{env.VAR}}` isn't present in the environment.
2. Call `create_or_update_api` **per request** in the batch, each conforming to `api.md`.

Follow `safeguard.md` throughout this step.

---

## Step 6 — Report, Then Never Auto-Run

After the batch:

- Summarize what was built (APIs + test cases created, env vars added, any skipped/renamed).
- **Ask before running anything** — never auto-execute a freshly built API or suite (`safeguard.md`).
- To continue, **re-confirm the next folder** at Step 4 before building it. The Step 3 convention ledger carries over unchanged unless the user amends it.

---

## Swagger / OpenAPI Specifics

When the source is an OpenAPI 3.x / Swagger 2.0 spec instead of a Postman collection, the mapping in Steps 2–4 changes as follows; everything else is identical:

- **Grouping (folders):** use the operation `tags` (or the first path segment when untagged) as the batch unit, in place of Postman folders.
- **API name:** prefer `operationId` when present (converted to `snake_case`); otherwise derive from `method` + the meaningful path segments (e.g. `GET /policies/{id}` → `get_policy_by_id`).
- **Endpoint / base URL:** `servers[].url` (OpenAPI) or `host` + `basePath` (Swagger) → `{{env.BASE_URL}}` per Step 3.1. Path templating `{param}` maps to test-case params / flow inputs, not hardcoded values.
- **Auth:** `components.securitySchemes` (or Swagger `securityDefinitions`) → the Step 3.2 auth decision (typically bearer → `{{env.TOKEN}}`).
- **Payload:** synthesize a sample body from the `requestBody` schema (`example`/`examples` first, else schema defaults) → the test-case `payload`.
- **Content type:** the operation's `requestBody` media type → the API `content_type` (`application/json`, `multipart/form-data`, …) per `api.md`.

---

## What This Rule Is Not

- It is not a substitute for `negotiation/api.md` — each built API still follows it. This rule only adds the parse-inventory-negotiate-conventions-once-then-batch layer on top.
- It does not apply to a single cURL / single endpoint (use `api.md`), nor to information-only requests ("list the folders", "show me the collection").
- It does not permit silent bulk creation. Every batch is confirmed; nothing is auto-run.

> **On any conflict between this file and `api.md` / `safeguard.md`, those rules win for the concern they own** (API structure → `api.md`; write/run safety → `safeguard.md`). This file owns only the bulk-import orchestration.
