# QA Flow — Dataset Binding (Data-Driven Testing)

A `dataset` object makes a **flow** or an **API test case** run once per row of an external dataset. The test logic is written once; the data comes from an Assets Manager file, a saved database query, or inline rows. Each row is reported as its own pass/fail.

---

## Where It Lives

| Binding point | Where the `dataset` object goes | What iterates |
|---|---|---|
| Flow | top level of the flow JSON, next to `inputs` | the whole flow runs once per row (fresh context per row) |
| API test case | inside the test case, next to `payload` | that test case runs once per row (one pytest item per row) |

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
  "on_row_failure": "continue"        // default; "stop" ends the set at the first failing row
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

Validate and inspect a binding with the **`preview_dataset`** MCP tool (dashboard: *Preview rows*) — it returns the columns, the first rows, the row ids and the total count. `list_dataset_sources` lists the tabular assets and saved query methods available.

---

## Using Row Values

| Where | Syntax |
|---|---|
| `payload`, `params`, `query_params`, expected values | `{{data.<column>}}` |
| `header_import.variable` | `data.<column>` (short form) |
| `context_import` | `data.<column>` |
| Python-code conditions / wait-until | `get_var('data.<column>')` |
| Declared flow input with the **same name** as a column | seeded automatically per row (`{{context.flow_input.<name>}}` keeps working) |

Rules:

- A string that is **exactly** `{{data.col}}` takes the column's native value (number, boolean, list); a placeholder inside a longer string is interpolated as text.
- CSV / XLSX cells are **strings** (empty cell → `""`); use `types` to coerce. JSON, inline and query rows keep their native types. `"expected_status": "{{data.status}}"` works when the column is an integer (or coerced with `types`).
- A missing column fails **that row** with a message naming the column and the available columns; the run continues unless `on_row_failure: stop`.
- `{{auto.*}}`, `{{env.*}}` and `{{context.*}}` keep working alongside `{{data.*}}`.

---

## Row Ids and Results

- Each row gets an id from an `_id`, `id` or `name` column (sanitised to `[A-Za-z0-9_-]`, de-duplicated) or `row_<n>`.
- Pytest items are named `<test_case>[<row_id>]` (API cases) / `<flow>[<row_id>]` (flows), so `-k <base name>` still selects the whole set and `QA_FLOW_ONLY_TEST_CASE` matches the base name.
- A data-driven flow's result context carries `dataset_rows` (one entry per row: `row_id`, `row`, `status`, `passed`, `steps_executed`, `errors`, `duration`) and `dataset_summary` (`rows_total`, `passed`, `failed`, `stopped_early`, `failed_row_ids`). `steps_executed` / `all_steps_passed` / `errors` (prefixed `[row_id]`) are kept for backward compatibility — `steps_executed` holds the first failing row's steps, or the last row's.
- Test-group reports list one test case per row (`<flow>[<row_id>]`) with the row attached; passing rows are trimmed and collapsed, failing rows keep their responses and are shown with their data.
- `expected_flow_execution_time` applies **per row**.

---

## Runtime Overrides and Edge Cases

- **Ad-hoc rows for one run:** `run_test_suite(suite, dataset_rows=[...])` (dashboard: *Override rows* in the Run dialog) sets `FLOW_DATASET_JSON`, which replaces the saved binding for that run.
- **Sub-flows:** inside a data-driven parent, a sub-flow runs **once with the parent's current row** (its own `dataset`, if any, is not iterated).
- **Dataset-bound API test case called from a flow step:** the flow's own row is used and the case's binding is ignored. Running such a case with no row at all (e.g. through the prerequisite runner) is a hard error with guidance.
- **Query-bound datasets** hit the database at pytest collection time. Keep them bounded (`limit`).
- **Large datasets:** more than 1000 rows (after `filter` / `limit`) is warned about, never blocked — `preview_dataset` returns a `warning`, the dashboard preview shows it, and runs log it and raise a `LargeDatasetWarning` in pytest. Suggest `limit` / `filter` when you see it.
- **Assets:** only `.csv`, `.xlsx`, `.xls`, `.json` assets are accepted (`.xls` is read through `xlrd`, a framework dependency — a project missing it gets an error naming `uv add xlrd`); duplicate CSV headers and a dataset of 0 rows are errors (a suite must never pass by running nothing).
