# QA Flow Test Group Negotiation Rule

## Scope
These rules apply to **any AI assistant** whenever a user requests the creation or update of a **test group** using the QA Flow MCP tools (`create_test_group`, `update_test_group`, `modify_test_group_items`).

A test group bundles existing **items** — `flow`, `test_suite`, and `query` — into one runnable, optionally release-tagged collection. The items must already exist; a test group never creates them.

---

## Core Principle — Negotiate Before You Build

**Never call `create_test_group`, `update_test_group`, or `modify_test_group_items` as the first response to a user request.**

**No tool calls before negotiation begins.** The FIRST response to a test-group request must be the Step 2 name-and-description question — **not** `list_test_groups`, `get_test_group_available_items`, `health_check`, or any other read-only tool. "Gathering context first" is the forbidden pattern itself. Context-gathering happens in Step 1, **after** the name question is answered.

Negotiation means: ask one question, wait for the answer, ask the next. The user drives the decisions; you surface the options.

---

## Step 1 — Gather Context from the MCP Server (After the Name Is Confirmed)

Only **after** the user has answered the Step 2 name/description question, query the server:

1. `list_test_groups` — does a similar group already exist? Is this a new group or an update?
2. `get_test_group_available_items` — which `flow`, `test_suite`, and `query` items can be added.
3. If a similar group exists, call `get_test_group` on it to understand existing patterns (items, order, execution config, release). Use this as **reference only** — never assume the new group should copy it.

Use this context to inform your questions, not to skip them.

---

## Step 2 — Ask One Question at a Time (Sequential, Strictly)

**Critical rules:**
- **One topic per message.** High-stakes decisions (name, items, plan confirmation) get their own question; **related low-priority settings are grouped into a single message** per the *Grouped low-priority parameters* rule in `ai-rules/reference/selection-format.md`. Never bundle unrelated concerns or the whole plan into one reply.
- **Never skip a question** because a similar group exists or because you think you have enough context.
- **Never present a summary or proposed plan** until every question below has been answered.
- **After each answer, ask only the next question — nothing else.**

Ask in this exact order, one message per question:

1. **Name and description** — Propose a concise `snake_case` name and a one-sentence description, then ask the user to confirm or change it. (This is the first response — **no tool calls** before it.)
2. **Which items, and in what order? — ONE grouped message.** Present the available items from `get_test_group_available_items` as a numbered list (laid out per the Selection format rule), grouped by type — e.g. `Flows: (1) login_flow  (2) checkout_flow   Suites: (3) test_login   Queries: (4) get_user_status`. Ask which to include **and** in what execution order (the order they reply in is the proposed order — confirm it in the plan). **If a needed flow, suite, or query does not exist**, it must be created first under its own rule (`ai-rules/negotiation/flow.md` for flows, `ai-rules/negotiation/api.md` for APIs/suites, `ai-rules/negotiation/query.md` for queries) — never invent an item name.
3. **Group settings — ONE grouped message: stop-on-failure + release + tags.** Do **not** ask these three one-by-one:

   ```
   Group settings (defaults shown — confirm or change any):
   - Stop on first failure? default: no (run every item)
   - Release tag (e.g. v1.2) for release comparison? default: none
   - Tags (e.g. smoke, regression) for filtering? default: none
   ```

   (`dry_run` stays `false` unless the user asks for a no-op validation run.) Accepting the defaults in one reply is the explicit confirmation of all three.

> **Selection format rule:** small fixed choice sets → an inline numbered menu on one line — `(1) a  (2) b  (3) c`. Long lists of existing items → a numbered table/list, one option per line with a short identifying detail; show at most **20** (tell the user more exist and to just name theirs); prefix rows with `- [ ]` when several can be picked. Full rule, examples, and native structured-UI guidance: read `ai-rules/reference/selection-format.md` before presenting your first list.

---

## Step 3 — Present a Proposed Plan and Ask for Confirmation

Only after **all questions above are answered**, present the full summary:

- **Group name** and description
- **Items in order**: each as `type → name` (e.g. `flow → login_flow`, `test_suite → test_checkout`)
- **Execution config**: `stop_on_failure`, `dry_run`
- **Release** tag — or "none"
- **Tags** — or "none"
- **Assumptions** you applied (list every default)

Then ask: **"Does this look right, or would you like to change anything before I build it?"**

Do not proceed until the user explicitly confirms. If the user requests a change, apply only that change and re-present for confirmation.

---

## Step 4 — Build

Only after explicit user confirmation:

1. Confirm **every item exists** (`get_test_group_available_items`). Create any missing flow/suite/query first under its own negotiation rule.
2. Call `create_test_group` with the agreed `name`, `description`, `items`, `release`, `tags`, and `execution_config`.
   - Pass each item as just `{ "type", "name" }` (add `config.parameters` only when the flow/query needs input values — see *Item Structure*). The server **auto-generates** `id`, `order`, `description`, and `config` for every item, so you never hand-build them.
   - **For updates:** call `get_test_group` first, modify the complete group object, then send it all back via `update_test_group` — it replaces the entire definition. Use `modify_test_group_items` for incremental add / reorder / remove.
3. Call `validate_test_group` before executing — catch missing or malformed items early.
4. Follow all rules in `ai-rules/safeguard.md`.

---

## Step 5 — After Creating the Group, Ask to Run It

After `create_test_group` / `update_test_group` completes successfully, **always** ask — as its own message, one question:

```
The test group "[group_name]" has been created successfully. Would you like to run it now?
```

- If **no** → stop and wait.
- If **yes** → do **not** execute immediately. First negotiate the run options as **ONE grouped message**:
  **Which environment, and background or foreground?** Call `list_environments` and present the available environments (laid out per the Selection format rule) so the user picks which one to run against — the chosen environment's `id` becomes `environment_id` on `execute_test_group`. Never pick an environment silently; if there is only one, confirm it rather than assuming. In the same message, ask whether to run in the **background** (`background=true`, the default — returns a `job_id` to poll with `get_run_status`; best for long-running groups) or in the **foreground** (`background=false` — waits and returns results inline; best for short groups).
- Only **after both are answered** → call `execute_test_group(group_name, environment_id, background)`. If run in the background, poll with `get_run_status`.
- **After the run completes**, summarize the result, then — as its own message, one question — **always offer to export it**: *"Would you like to export these results? Supported formats: JSON, CSV, HTML, PDF."* (laid out per the Selection format rule). If the user picks a format → `export_test_group_results(group_name, execution_id, format[, save_to])` — this is the report dashboard's 📤 export action, so follow `ai-rules/negotiation/report-dashboard.md`. If the user declines, stop.
- Do **not** run automatically, export automatically, or combine any of these questions with another message.

---

## Established Patterns — Apply These When Building

### Naming Conventions
- Use `snake_case` group names that reflect purpose: `smoke_regression_suite`, `release_v2_checks`, `nightly_full_run`.

### Item Structure
- Pass each item as `{ "type": "flow" | "test_suite" | "query", "name": "<exact_name>" }`. `name` must match exactly what `get_test_group_available_items` returns — a wrong name fails validation.
- **`id`, `order`, `description`, and `config` are auto-generated server-side — do NOT construct them.** `create_test_group`, `update_test_group`, and `modify_test_group_items` (add/reorder) normalize every item: each gets a unique `id`, an `order` by position, and `config` defaulted to `{ "parameters": {} }`. The executor reads `item['config']` directly, so this normalization is what prevents the run-time `KeyError: 'config'` that minimal `{type, name}` items used to trigger.
- **Only supply `config.parameters`** when a flow/query needs input values — populate it from the item's `inputs` array in `get_test_group_available_items`. Anything you provide is preserved; anything you omit is defaulted.
- **Static parameter values:** the values you put in `config.parameters` feed a flow's declared inputs, so per-run test data belongs here. But if a value is environment-specific or a secret (base URLs, tenant IDs, tokens), don't hardcode it in the group — it should be an env var referenced inside the flow itself (`{{context.env.X}}`), not pinned per test group. Flag such cases to the user instead of baking the literal into the group definition.

The stored shape (after server normalization) looks like:

```json
{
  "id": "item_<auto>",
  "type": "flow" | "test_suite" | "query",
  "name": "<exact_name>",
  "order": 0,
  "description": "",
  "config": { "parameters": {} }
}
```

### Execution Config
- `execution_config` keys: `stop_on_failure` (bool, default `false`), `dry_run` (bool, default `false`).
- `timeout` is supplied at execution time (`execute_test_group`, default `3600` seconds), not stored in the group.

### Update Semantics
- `update_test_group` replaces the **entire** group object — always `get_test_group` first, modify, send complete.
- `modify_test_group_items` is for incremental changes: `action='add'` (with `item`), `'reorder'` (full items list), `'remove'` (with `item_id`).

---

## What This Rule Is Not

- It is not a rigid questionnaire. Questions are adapted to the request — but they are never skipped.
- It does not block quick edits where all details are already explicitly stated by the user.
- It does not apply to information-only requests (e.g. "list the test groups", "show me group X").
- A similar existing group is **reference material only** — never a substitute for asking the user what they want.
