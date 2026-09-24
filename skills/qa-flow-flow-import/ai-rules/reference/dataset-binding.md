# QA Flow — Dataset Binding (Data-Driven Testing)

A `dataset` object makes a **flow**, an **API test case** or a **saved query** run once per row of an external dataset. The test logic is written once; the data comes from an Assets Manager file, a saved database query, or inline rows. Each row of a flow / test case is reported as its own pass/fail; a saved query is **execute-only** (see *Saved Queries* below).

---

## Where It Lives

| Binding point | Where the `dataset` object goes | What iterates |
|---|---|---|
| Flow | top level of the flow JSON, next to `inputs` | the whole flow runs once per row (fresh context per row) |
| API test case | inside the test case, next to `payload` | that test case runs once per row (one pytest item per row) |
| Saved query | `code_generator/DataSources/query_definitions/<name>.json`, under `dataset` | the query runs once per row, in the dashboard only (execute-only) |

Per-step dataset binding is **not** supported. A flow or test case without `dataset` behaves exactly as before.

---

## Schema

```json
"dataset": {
  "source": "asset",                  // "asset" | "query" | "inline"
  "ref": "users.csv",                 // asset: filename or key · query: saved query METHOD name
  "sheet": "Sheet1",                  // asset xlsx / xls only, optional (first sheet by default)
  "params": {"client_id": "{{env.CLIENT_ID}}", "hours": "{{flow_input.hours}}"},   // query only, optional
  "rows": [{"email": "a@x.com"}],     // inline only
  "limit": 200,                       // optional — first N rows
  "filter": {"country": "LB"},        // optional — keep rows whose column equals the value
  "types": {"age": "int"},            // optional — per-column coercion: int | float | bool | json | str
  "on_row_failure": "continue",       // default; "stop" ends the set at the first failing row
  "execution": "sequential",          // flows: default, one row after the other; "parallel" = all rows at once
  "max_parallel": 10                  // parallel only: rows running at the same time, 1-50 (default 10)
}
```

| Field | Required | Notes |
|---|---|---|
| `source` | **yes** | `asset` (CSV / XLSX / XLS / JSON uploaded through the Assets Manager), `query` (saved query rows), `inline` |
| `ref` | asset, query | Asset **filename or key** (`upload_asset` / `manage_asset`), or the saved query **method name** (see `list_dataset_sources`) |
| `sheet` | no | XLSX / XLS only |
| `params` | no | Query kwargs; `{{env.VAR}}` and `{{flow_input.name}}` are resolved before the query runs |
| `rows` | inline | Non-empty list of row objects |
| `limit` / `filter` / `types` | no | Applied in that order: filter → limit → types. `filter` keeps a row only when it **has** the column and the value matches: a JSON `true`/`false` matches the words `types: bool` accepts (`"true"`, `"Yes"`, `""` for false…), a number matches a numeric cell (`30` ↔ `"30"`), `null` matches only a null value, anything else compares as text |
| `on_row_failure` | no | `continue` (default) or `stop` — a stop ends only that set: the same test case name in another suite is unaffected |
| `execution` | no | **Flows only.** `sequential` (default): row 1, then row 2, then row 3 — each row starts when the previous one ends. `parallel`: every row starts at once in the same pytest process and the results are assembled into one report when the last row ends. Pick parallel only when the rows don't depend on each other (no shared account, no ordering) |
| `max_parallel` | no | Parallel only: how many rows run at the same time (1–50, default 10); the rest wait for a free slot |

Validate and inspect a binding with the **`preview_dataset`** MCP tool (dashboard: *Preview rows*) — it returns the columns, the first rows, the row ids and the total count. `list_dataset_sources` lists the tabular assets and saved query methods available.

---

## Using Row Values

**Columns reach a request through the `mapping` (next section), never through a typed placeholder.** Flow steps and API test cases keep their literal values; the mapping names which column replaces which literal, and every unmapped key sends its own value on every row. A run without data (▶ Run) sends the literals as they are. Do not write `{{data.<column>}}` into a step or a test case — placeholders found in older definitions still resolve (and `list_data_placeholders` finds them so they can be turned back into literals), but nothing new should add one.

| Where | Syntax |
|---|---|
| `payload`, `params`, `query_params`, `headers` | a `mapping` row (`column → destination.path`) — legacy `{{data.<column>}}` still resolves |
| `header_import.variable` | `data.<column>` (short form) |
| `context_import` | `data.<column>` |
| Python-code conditions / wait-until | `get_var('data.<column>')` |
| Declared flow input with the **same name** as a column | seeded automatically per row (`{{context.flow_input.<name>}}` keeps working) |

Rules:

- A string that is **exactly** `{{data.col}}` takes the column's native value (number, boolean, list); a placeholder inside a longer string is interpolated as text.
- CSV / XLSX cells are **strings** (empty cell → `""`); use `types` to coerce. JSON, inline and query rows keep their native types. `"expected_status": "{{data.status}}"` works when the column is an integer (or coerced with `types`).
- **Priority — the step key wins, then the test case.** For every payload/params key: if the flow step sets it, that value is sent — `{{data.*}}` resolves to the row, a literal is sent as-is — to **every** selected test case (`api_call` and `wait_until`, one or many test cases). Only keys the step does not set come from the test case definition, where `{{data.*}}` also resolves. So to give one test case its own column (e.g. a negative case's `invalid_username`), map it with a `level: "test_case"` mapping row for that case and leave the key out of the step.
- A missing column fails **that row** with a message naming the column and the available columns; the run continues unless `on_row_failure: stop`.
- **Parallel rows** (`execution: parallel`): each row runs on its own copy of the flow with fresh API clients and its own shared session, so rows never see each other's cookies or context. The flow is **one** pytest item (not one per row); its `dataset_rows` still list every row in dataset order, and `flow_elapsed_time` is the wall-clock time of the whole batch. Each row's log is printed as one block when that row finishes. `on_row_failure: stop` only skips the rows that have not started yet — rows already running finish. The context carries `dataset_execution: {"mode": "parallel", "max_parallel": N}` (`{"mode": "sequential"}` otherwise).
- A sub-flow never runs its own rows in parallel inside a parent: the parent owns the row loop (and its execution mode).
- `{{auto.*}}`, `{{env.*}}` and `{{context.*}}` keep working alongside `{{data.*}}`.

---

## Mapping — Columns Without Typing Placeholders

A binding may carry a `mapping`: which column feeds which value the user left as a **literal**. Nobody has to type `{{data.*}}` for those keys; the dashboard's Run dialog infers the rows (🔮, with a reason each), the user confirms them, and they are saved on the binding so CLI / pytest / CI runs behave exactly like the dashboard run.

```json
"mapping": [
  {"column": "user_name", "step": "step_2", "level": "step",
   "destination": "payload", "path": "username"},
  {"column": "mail", "step": "step_2", "level": "test_case", "test_case": "login_test",
   "destination": "payload", "path": "credentials.email"}
],
"dismissed": ["step_2|test_case:invalid_login_test|payload|username"]
```

| Field | Values |
|---|---|
| `column` | a column of the dataset — missing at run time fails that row, naming the mapping |
| `destination` | `payload` · `params` · `query_params` (DB steps) · `headers` |
| `path` | dot path into that destination, **no array indices** |
| `step` / `level` | flow bindings only: `step` rewrites the step's own override (so every selected test case gets it), `test_case` needs `test_case` and touches only that one |
| `dismissed` | mapping keys (`step\|level[:case]\|destination\|normalized path`) never to propose again |

Rules:

- **Precedence:** an explicit `{{data.col}}` wins, then the mapping, then the literal. A value already wired — `{{context.*}}`, `{{env.*}}`, `{{auto.*}}`, `header_import` — is never mapped, and credential headers (auth / CSRF / API key / session) are off limits; they belong to prerequisite inference.
- A **test-case binding**'s rows carry only `column` / `destination` / `path`.
- `infer_dataset_mapping(kind, name)` proposes rows — origin (`exact` / `path` / `synonym` / `negative_prefix`), reason, current value, and `enabled` (rows for a negative test case come back unticked) — and flags saved rows as `stale` when their column or path is gone. It **writes nothing**: save the rows you keep through `update_flow` / `create_or_update_api`.
- A **sub-flow** is handed the parent's row, never its mapping. Its own mapping therefore resolves against the **parent's** columns; `infer_dataset_mapping` and `validate_flow` report a sub-flow whose mapping reads a column the parent's dataset does not have.

---

### Values no column matches

Inference lists **every** literal it may claim, including the ones no column matches: those come back with `column: null` and `unmatched: true`, unticked, so a column can be assigned by hand. Without them a DB step's `query_params` — usually one or two named parameters — could never be mapped at all.

The dashboard keeps the table readable: unmatched **`query_params`** rows stay in the main table (a query step's inputs are normally the point of the run), everything else folds into a *"N values no column matched"* group under it, closed until clicked. Picking a column ticks the row and it saves like any other.

## Running Without the Data

A data-driven flow can also run **once, on its own configuration** — the binding ignored, the mapping not applied, nothing saved:

- **Row mode for one run:** `run_test_suite(suite, dataset_execution="parallel", dataset_max_parallel=5)` (dashboard: 📊 Run with data → *How the rows run*) overrides the saved `execution` / `max_parallel` for that run only (`FLOW_DATASET_EXECUTION` / `FLOW_DATASET_MAX_PARALLEL`). A no-data run ignores it.
- `run_test_suite(suite, use_dataset=false)` (dashboard: the ▶ **Run** button on the flow card — it never uses a saved dataset; 📊 **Run with data** is the per-row path, where the dataset itself is chosen).
- Every `{{data.*}}` still wired by hand needs a literal: `data_values={"username": "..."}`. List them first with `list_data_placeholders`, which also gives each one's locations and a value from the first row. Without a literal the run is **refused, naming them** — no request ever carries a raw `{{data.x}}` string.
- Sub-flows count: a no-data run skips the sub-flow's own dataset as well, so `{{data.*}}` typed inside a sub-flow are leftovers of the parent run — `list_data_placeholders` on the parent lists them under the calling step (`step_1 → inner_flow · …`, flagged `in_subflows_only`) and `data_values` must cover them. Removing the parent's binding never rewrites those.
- The result is marked `dataset_skipped` and has no `dataset_rows`, so a green single run is never mistaken for "all rows passed".
- Permanent version: **remove the binding** (`update_flow` without `dataset`), which turns those placeholders into fixed values (`placeholder_values`) and regenerates. A sub-flow fed by a data-driven parent is the one case where keeping a placeholder makes sense.

---

## Row Ids and Results

- Each row gets an id from an `_id`, `id` or `name` column (sanitised to `[A-Za-z0-9_-]`, de-duplicated) or `row_<n>`.
- Pytest items are named `<test_case>[<row_id>]` (API cases) / `<flow>[<row_id>]` (flows), so `-k <base name>` still selects the whole set and `QA_FLOW_ONLY_TEST_CASE` matches the base name.
- A data-driven flow's result context carries `dataset_rows` (one entry per row: `row_id`, `row`, `status`, `passed`, `steps_executed`, `errors`, `duration`) and `dataset_summary` (`rows_total`, `passed`, `failed`, `stopped_early`, `failed_row_ids`). `steps_executed` / `all_steps_passed` / `errors` (prefixed `[row_id]`) are kept for backward compatibility — `steps_executed` holds the first failing row's steps, or the last row's.
- Test-group reports list one test case per row (`<flow>[<row_id>]`) with the row attached; passing rows are trimmed and collapsed, failing rows keep their responses and are shown with their data.
- `expected_flow_execution_time` applies **per row**.

---

## Runtime Overrides and Edge Cases

- **Ad-hoc rows for one run:** `run_test_suite(suite, dataset_rows=[...])` (dashboard: 📊 Run with data → *Inline rows* + *Use for this run only*, which the dashboard sends as a run-only `dataset_binding` the server loads into rows) sets `FLOW_DATASET_JSON`, which replaces the saved binding for that run.
- **Sub-flows:** inside a data-driven parent, a sub-flow runs **once with the parent's current row** (its own `dataset`, if any, is not iterated).
- **Dataset-bound API test case called from a flow step:** the flow's own row is used and the case's binding is ignored. Running such a case with no row at all (e.g. through the prerequisite runner) is a hard error with guidance.
- **Query-bound datasets** hit the database at pytest collection time. Keep them bounded (`limit`).
- **Large datasets:** more than 1000 rows (after `filter` / `limit`) is warned about, never blocked — `preview_dataset` returns a `warning`, the dashboard preview shows it, and runs log it and raise a `LargeDatasetWarning` in pytest. Suggest `limit` / `filter` when you see it.
- **Assets:** only `.csv`, `.xlsx`, `.xls`, `.json` assets are accepted (`.xls` is read through `xlrd`, a framework dependency: `qa-flow update` adds it to an existing project's `pyproject.toml` — then `uv lock && uv sync`; a project still missing it gets an error naming `uv add xlrd`); duplicate CSV headers and a dataset of 0 rows are errors (a suite must never pass by running nothing).


---

## One API Test Case, Once Per Row (Manage API Endpoints)

A test case's own binding can also be run **from the API card**, without a flow around it: 📊 *Run with data* next to ▶ Run.

- **Scope is one test case.** The dialog picks the case, the dataset (asset / saved query / inline, with a preview), the column → value mapping and how the rows run. The binding is saved on that case unless *Use for this run only* is ticked; other cases of the same API keep their own.
- **▶ Run (and 🔑) on an API card send the case's own values**: every case runs once, a saved binding is skipped (`use_dataset: false` on `POST /api/apis/<name>/run`; MCP `run_api(use_dataset=False)`). Rows feed a run only through 📊 — and through the generated suite and CI, which still iterate a saved binding.
- **Mapped key → column, unmapped key → the case's value.** The case holds literals only; no `{{data.*}}`.
- **Auth per row.** A saved **prerequisite template** can be picked in the dialog; its chain runs again for every row, so each row gets its own login. An ad-hoc chain is not offered here — save it as a template first (the 🔑 dialog builds one).
- **Every row is its own process.** Sequential and parallel both run one pytest per row (`dataset_execution`, `dataset_max_parallel` 1–50, default 10); the row is handed over in `FLOW_DATASET_JSON` and the run-only binding in `QA_FLOW_CASE_DATASET_JSON`, which overlays what the case has saved — that is how a case with **no** binding can run against a dataset chosen in the dialog. Start-up costs about a second per row, so parallel pays off on slow endpoints or many rows.
- **MCP:** `run_api(api_name, test_case="<one case>", dataset=…, dataset_mapping=[…], dataset_execution=…, dataset_max_parallel=…)`. `test_case` must name ONE case — a data-driven run cannot target them all — and nothing passed to the tool is saved (`create_or_update_api` persists a binding). The result carries `dataset_rows` and `dataset_summary`; only failing rows keep their output.
- **`on_row_failure: stop`** ends the set at the first failing row; in parallel mode it skips only the rows that have not started.

---

## In a Test Group — Per-Item Override

A data-driven item in a test group runs on the dataset saved on its flow / API test case, once per row, reported as `<flow>[<row_id>]` / `<case>[<row_id>]` with a `dataset_summary`; **any failed row fails the item** (so `stop_on_failure` stops the group after it). The item's `config.dataset_override` changes its data **for that group only** — the flow / test case is untouched:

```json
"config": {
  "parameters": {},
  "dataset_override": {
    "dataset": {"source": "inline", "rows": [...]},   // another dataset (keeps the saved mapping — same column names)
    "mapping": [ ... ],                                // another mapping (optional)
    "execution": "parallel", "max_parallel": 5          // how the rows run
  }
}
```

`{"use_dataset": false, "data_values": {...}}` runs the item once on its own values. Every key is optional. On a `test_suite` (API) item it needs `selection_mode: "specific"` with exactly **one** selected test case; an item selecting several cases runs each bound case's rows through pytest and takes no override. Saves with a malformed override are rejected; `validate_test_group` reports it. Saved queries cannot be group items (execute-only).

## In a Load Test — One Row per Virtual User

A load test does **not** run every row per iteration. Rows are loaded once at test start and each Locust user is pinned to one row — user *i* uses row *i mod N* for all its iterations (one account per user). More users than rows → rows are shared, and `preview_load_test_dataset` / the enqueue response carry a warning; fewer users → the extra rows go unused. The row source is the item's `dataset_override` (a group item's, or `start_load_test(..., dataset_override=…)` for a single flow / API target, with `test_case` naming the API's one case), else the saved binding. Row `execution` mode is ignored — the users are the concurrency.

## Saved Queries (SQL and Mongo)

A saved query can carry the same `dataset` object, in a sidecar definition next to the generated method (`query_definitions/<name>.json`). Running it iterates the rows and collects **results**, not verdicts.

- **Mapping instead of placeholders.** A query has named parameters and no stored values, so `{{data.*}}` is never written into a query. The mapping's rows are bare — `{"column": "user_name", "destination": "query_params", "path": "<parameter name>"}` — one per parameter, and `path` is a parameter name, never a dot path. Columns are matched to parameters **by name** (case and separators ignored, plus the synonym table); a parameter nothing matches is left for the user to pick.
- **Execute-only.** There are no assertions and no pass/fail: each row reports its returned rows (capped for display) or the error it raised, and a failing row never stops the others. A data-driven query therefore cannot be put in a test group, a report, a release or a CI run — for that, wrap the query in a one-step flow, which is fully data-bound.
- **Where it runs.** The dashboard's Query Builder (▶ Run for one execution on the typed values, 📊 Run with data for one per row) and `run_saved_query`. No pytest suite is generated for a query.
- **MCP:** `run_saved_query(query_name, kind, parameters, dataset=…, dataset_mapping=[…], dataset_execution="sequential"|"parallel", max_parallel=…)`. Omit `dataset` to use the one saved on the query; nothing passed to the tool is ever saved.
- **Parallel rows** each get their own database connection, closed when the run ends. The singleton router is untouched, so `max_parallel` (default 10, max 50) is bounded by the database's own connection limit.
- **Types:** CSV cells are strings. A parameter annotated `int` / `float` / `bool` is cast from the string, and an empty value for an optional parameter becomes `None`; anything else needs a `types` entry on the binding.
- A query **cannot feed itself**: a binding with `source: "query"` whose `ref` is the query being run is refused.
