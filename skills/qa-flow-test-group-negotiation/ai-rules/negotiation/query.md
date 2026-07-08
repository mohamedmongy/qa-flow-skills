# QA Flow Query Negotiation Rule

## Scope
These rules apply to **any AI assistant** whenever a user requests the creation or update of a **saved database query** using the QA Flow MCP tools (`save_query`, `update_query`, `duplicate_query`, `delete_query`).

A saved query is a reusable, parameterized DB method that flows reference from a `query` step (`query_method` + `query_params`) and that `run_saved_query` can execute directly. There are three **kinds**:

| Kind | Database | Stored as | Use it for |
|---|---|---|---|
| `sql` | PostgreSQL (e.g. `omni_postgres`) | a Python method in `user_queries/postgres_queries.py` | any SQL — SELECT/INSERT/UPDATE/DELETE, raw SQL, DO/CALL blocks |
| `mongo` | MongoDB | a Python method in `user_queries/mongo_queries.py` **and** a JSON definition in `code_generator/DataSources/query_definitions/` | MongoDB find/findOne/count/aggregate/update |
| `definition` | MongoDB | a JSON definition only (no Python method) | a declarative Mongo query definition without generated code |

> This rule governs **authoring** queries. Pure read/inspection (`list_queries`, `get_query`) is not gated. `test_query`, `validate_sql`, and `run_saved_query` are part of the build/smoke-test workflow below — they are used **during** Steps 1 and 4, never as the opening move.

---

## Core Principle — Negotiate Before You Build

**Never call `save_query`, `update_query`, `duplicate_query`, or `delete_query` as the first response to a user request.**

**No tool calls before negotiation begins.** The FIRST response to a query-creation request must be the Step 2 name-and-kind question — **not** `list_queries`, `list_databases`, `get_db_schema`, `get_db_status`, `health_check`, or any other read-only tool. "Gathering context first" is the forbidden pattern itself. Context-gathering happens in Step 1, **after** the name question is answered.

Negotiation means: ask one question, wait for the answer, ask the next. The user drives the decisions; you surface the options.

---

## Step 1 — Gather Context from the MCP Server (After the Name Is Confirmed)

Only **after** the user has answered the Step 2 name/kind question, query the server:

1. `list_queries` — does a query with this name (or one doing the same job) already exist? Is this a new query or an update?
2. `list_databases` / `get_db_status` — which databases are connected, and which are SQL (PostgreSQL) vs MongoDB. The chosen `database` name must match exactly.
3. `get_db_schema(db_name)` — discover the real tables/columns (SQL) or collections/fields (Mongo) so the query targets things that actually exist. **Never invent a table, column, or collection name.**
4. If a similar query exists, call `get_query` on it (with the right `kind`) to understand the existing pattern — **reference only**, never assume the new query should copy it.

Use this context to inform your questions, not to skip them.

---

## Step 2 — Ask One Question at a Time (Sequential, Strictly)

**Critical rules:**
- **One topic per message.** High-stakes decisions (name+kind, database, destructive intent, plan confirmation) get their own question; **related low-priority parameters are grouped into a single message** per the *Grouped low-priority parameters* rule in `ai-rules/reference/selection-format.md`. Never bundle unrelated concerns or the whole plan into one reply.
- **Never skip a question** because a similar query exists or because you think you have enough context.
- **Never present a summary or proposed plan** until every question below has been answered.
- **After each answer, ask only the next question — nothing else.**

Ask in this exact order, one message per question (skip a question only when the user has already stated that detail unambiguously):

1. **Name & kind** — Propose a concise `snake_case` method name (e.g. `get_user_by_email`, `count_active_subscriptions`) and ask whether this is a **SQL** query (PostgreSQL), a **Mongo** query (generates a Python method), or a **Mongo definition** (JSON only). (This is the first response — **no tool calls** before it. The name must match `^[a-zA-Z_][a-zA-Z0-9_]*$`.)
2. **Which database?** Present the connected databases from `list_databases` as a numbered list (laid out per the Selection format rule) and ask which one. The name becomes the query's `database`. **If the intended database is not connected**, stop and surface that — never guess a connection name.
3. **Target & operation — ONE grouped message.**
   - *SQL:* which table(s)/schema (confirm against `get_db_schema`), and does the statement **read or write**? (`query_type` is **ALWAYS `raw_sql`** — do not offer or pick any other SQL type. `raw_sql` routes through `execute_raw_sql`, which handles **every** PostgreSQL statement: `SELECT` (returns rows), `INSERT`/`UPDATE`/`DELETE` (returns rowcount), CTEs, window functions, subqueries, `CALL` stored procedures, `DO $$…$$` anonymous blocks, and DDL. The `return_type` is what shapes the result, not the query type — so don't ask "which SQL type", just the read-or-write question for the destructive-write guard.)
   - *Mongo:* which `collection`, and which `operation`: `(1) find  (2) findOne  (3) count  (4) aggregate  (5) update  (6) assert`.
   - **If the statement mutates data (any SQL write — INSERT/UPDATE/DELETE/CALL/DO, or Mongo `update`)** — flag it explicitly now; it is a write against a real database (see *Destructive-write guard*).
4. **The query body** —
   - *SQL:* ask for the `sql_query` (or derive it from what the user described). Use placeholders for every dynamic value — `%s` (positional) or `%(name)s` (named), consistently within one statement.
   - *Mongo:* ask for the `filter` (and `projection`), or the `pipeline` for `aggregate`. Use `"{param_name}"` placeholders for dynamic values.
   - **For every static literal** (client IDs, tenant IDs, status IDs, usernames, dates, secrets) apply *Static Values → Parameters or Env Vars* below — do **not** hardcode it. Several pending literals are presented together in one table-style message, each picking its home.
5. **Parameters & return type — ONE grouped message.** Present **all** placeholders in a single table-style question — for each, the parameter `name` (snake_case, valid Python identifier), `type`, and whether it is `required` (the count of parameters must match the count of placeholders) — and, in the same message, the return type:
   - *SQL* (`return_type`): `(1) record` (first row as dict) `(2) array` (all rows) `(3) field` (single value of first row) `(4) boolean` (rowcount > 0) `(5) count` (the count value).
   - *Mongo / definition*: `document` (find/findOne), `count`, `field`, or `boolean` — derived from the operation.

   ```
   The SQL has 2 placeholders — confirm each (name : type : required?) and the return type:
   - %s #1 → user_email : str : required?
   - %s #2 → status_id : int : required?
   Return type: (1) record  (2) array  (3) field  (4) boolean  (5) count
   ```

> ### ⚠️ Static Values → Parameters or Env Vars
> A dynamic or environment-specific literal in a filter/SQL is a prompt to ask, not a value to bake in:
> - **Varies per call / is test data** (user IDs, names, GUIDs, dates) → make it a **parameter** (placeholder + a `parameters` entry). When the query is used in a flow step, the value is supplied via `query_params` (e.g. `{{context.env.X}}`, `{{context.flow_input.X}}`, `{{context.step.field}}`).
> - **Environment-specific / shared config / secret** (base IDs, tenant IDs, tokens) → reference an **env var** `{{env.VAR_NAME}}` (resolved at test/run time when an `environment_id` is set), or pass it as a parameter from the flow's env context.
> - **Genuinely constant for every environment** → only then keep it literal, after the user confirms.

> **Selection format rule:** small fixed choice sets → an inline numbered menu on one line — `(1) a  (2) b  (3) c`. Long lists of existing items (including databases) → a numbered table/list, one option per line with a short identifying detail; show at most **20** (tell the user more exist and to just name theirs); prefix rows with `- [ ]` when several can be picked. Full rule, examples, and native structured-UI guidance: read `ai-rules/reference/selection-format.md` before presenting your first list.

> ### ⚠️ Destructive-write guard
> If the query mutates data (SQL `insert`/`update`/`delete`/`raw_sql` that writes, or Mongo `update`), say so plainly in the plan, name the table/collection affected, and confirm the user intends a write — and against which environment/database. Treat it with the care of any outward-facing action.

---

## Step 3 — Present a Proposed Plan and Ask for Confirmation

Only after **all questions above are answered**, present the full summary:

- **Method name**, **kind** (`sql` / `mongo` / `definition`), and **database**
- **Target**: table(s)/schema (SQL) or collection (Mongo)
- **Operation / query_type** and the **query body** (the SQL, or the filter/projection/pipeline)
- **Parameters**: each as `name : type (required|optional)` and which placeholder it binds
- **Return type**
- **Side effects you flagged** — e.g. *"this UPDATE writes to `billing.client_account`"*, or *"raw_sql DO block / stored-procedure call"*
- **Static values promoted** — for each literal, state whether it became a parameter, an env var, or was intentionally kept literal
- **Assumptions** you applied (list every default)

Then ask: **"Does this look right, or would you like to change anything before I build it?"**

Do not proceed until the user explicitly confirms. If the user requests a change, apply only that change and re-present for confirmation.

---

## Step 4 — Build

Only after explicit user confirmation:

1. **SQL only:** call `validate_sql(database, sql_query, parameters, return_type)` first — it checks syntax (EXPLAIN / sqlparse) and that the generated Python method compiles. Fix any error before saving.
2. Call `save_query` with the agreed parameters:
   - **`kind='sql'`** → `database`, `sql_query`, **`query_type='raw_sql'` (always)**, `return_type`, `parameters`.
   - **`kind='mongo'`** → `database`, `collection`, `operation`, `filter` (and `projection`, or `pipeline` for aggregate), `parameters`.
   - **`kind='definition'`** → `method_name`, `collection`, `operation`, `filter`, `projection`, `return_type`, `parameters` (JSON definition only, no Python method).
   - `parameters` format: `[{ "name": ..., "type": ..., "required": true|false }]`.
   - **For updates:** use `update_query` (`kind='sql'` → pass `database`/`sql_query`/`return_type`/`parameters`, optional `new_name`; `kind='mongo'` → pass the full replacement method `code`).
3. Do **not** smoke-test automatically — saving is where the build stops. Offer the smoke test as its own question in Step 5. (When the user accepts, `run_saved_query` / `test_query` results are capped at ~50 rows with `total_rows` preserved.)
4. Follow all rules in `ai-rules/safeguard.md` — **never** hand-edit `user_queries/*.py` or the `query_definitions/*.json`; the Query Builder owns those files.

---

## Step 5 — After Creating the Query, Ask What's Next

After `save_query` / `update_query` completes successfully, **always** ask — as its own message, one question:

```
The query "[method_name]" has been created successfully. Would you like me to smoke-test it now with sample parameter values?
```

- If **yes** → `run_saved_query` with the values the user provides (or `test_query` for an ad-hoc run). Report the rows returned.
- If **no** → stop and wait. Note it can now be wired into a flow via a `query` step (`query_method: "[method_name]"`) following `ai-rules/negotiation/flow.md`.
- Do **not** run automatically, and do **not** combine this question with any other message.

---

## Established Patterns — Apply These When Building

### Naming Conventions
- `snake_case`, valid Python identifier, descriptive of the action: `get_user_by_email`, `count_active_subscriptions`, `update_client_balance`.
- Parameter names are also `snake_case` valid identifiers.

### SQL is always `raw_sql`
- Every SQL query uses `query_type='raw_sql'` — it executes via `execute_raw_sql(sql, params)`, which the adapter documents as supporting "any valid PostgreSQL SQL statement": SELECT, INSERT/UPDATE/DELETE, `CALL`, `DO $$…$$`, CTEs, window functions, subqueries, DDL. There is no case that needs `select`/`insert`/`update`/`delete` instead.
- `execute_raw_sql` returns a `List[Dict]` for statements that produce rows (SELECT / `RETURNING`) and an `int` rowcount for commands; the `return_type` then shapes the final result. DO blocks that raise *"no results to fetch"* are treated as success.

### SQL parameter binding (PostgreSQL)
- Use **positional `%s`** placeholders, bound from a **tuple** in declaration order — this is the `raw_sql` convention (`params = (a, b, c)`). Example: `WHERE name = %s` → `params = (name,)`.
- **Required vs optional:** required params become positional method args; optional params get a default (`= None`) and are ordered after required ones.
- **Type conversion:** the declared type drives coercion — `int`/`INTEGER` → `int(value)`, `float`/`DECIMAL` → `float`, `bool`/`BOOLEAN` → truthy, empty string for an optional param → `None`. Allowed types include `str`/`VARCHAR`/`TEXT`, `int`/`INTEGER`/`BIGINT`, `float`/`NUMERIC`/`DECIMAL`, `bool`/`BOOLEAN`, `DATE`/`TIMESTAMP`, `dict`/`JSON`/`JSONB`, `list`/`ARRAY`.

### SQL return types
`record` (first row dict | `None`) · `array` (list of row dicts) · `field` (single value of first row) · `boolean` (`rowcount > 0`) · `count` (the count value).

### Mongo filter parameters
- Placeholders in the filter use `"{param_name}"` (quoted); the generator replaces `"{param_name}"` with the bare variable. Example: `{"email": "{user_name}"}` → `filter_query = {"email": user_name}`.
- Operations map to helpers: `find`/`findOne` (with `projection`), `count`, `aggregate` (with `pipeline`), `update`, `assert` (fails if no document matches).

### Env vars & placeholders
- Query **values** support `{{env.VAR_NAME}}` and `{{env.VAR_NAME|default}}` — resolved by `test_query` / runtime when an `environment_id` is set (`VAR_NAME` must be uppercase). They are **not** resolved at definition time.

### Result access from a flow step
- A `query` step exports under the **`result`** prefix (not `response`): `result.<field>` for a single record, `result[0].<field>` for arrays. After export it's referenced as `{{context.<step>.result.<field>}}`.

### Update & delete semantics
- **`update_query`** replaces the query — for SQL pass the full new `sql_query`/`parameters`; for Mongo pass the full method `code`. Rename via `new_name`.
- **`delete_query` is destructive** and **fails if the query is still referenced** by any flow or test group — it returns the list of users (`used_in`). Remove the references first, or pick a different approach. Never force-delete a referenced query without telling the user what depends on it.

---

## What This Rule Is Not

- It is not a rigid questionnaire. Questions are adapted to the request — but they are never skipped, and static literals are never silently hardcoded.
- It does not block a quick edit where the user has **already stated every detail explicitly** ("save a SQL select on `omni_postgres`: `SELECT id FROM users WHERE name = %s`, param `name` str, return record") — but you still confirm the database exists, validate SQL, and echo the plan before saving.
- It does not apply to information-only requests (e.g. "list the queries", "show me query X") — those use the read tools directly.
- A similar existing query is **reference material only** — never a substitute for asking the user what they want.
