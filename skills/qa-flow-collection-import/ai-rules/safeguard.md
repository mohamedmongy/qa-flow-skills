# QA Flow MCP Safeguard Rules

## Scope
These rules apply whenever using the QA Flow MCP tools to create, update, or debug flows, APIs, queries, test cases, or test groups.

---

## Rule 1 — Always use MCP tools to create and modify files

All API definitions, flows, queries, test cases, and data sources **must be created and updated exclusively through the QA Flow MCP tools** (e.g., `create_or_update_api`, `create_flow`, `update_flow`, `save_query`).

**Never** create or modify generated files directly by writing or editing `.py`, `.json`, or test suite files on disk — even if it seems like a quick fix.

**Why:** The generator owns those files. Direct edits get silently overwritten the next time the tool regenerates the suite, and they mask real root causes.

---

## Rule 2 — Diagnose root cause before fixing

When a test or flow fails, investigate **why** the tool produced the wrong output before touching anything.

Steps:
1. Read the error carefully and identify which layer failed (data source, API definition, flow step config, assertion, header injection, etc.).
2. Use `get_api_definition`, `get_test_cases`, `get_flow`, or `get_query` to inspect the current state.
3. Compare against a known-working equivalent (e.g., another flow that does the same thing).
4. Fix the root cause by updating the definition through the correct MCP tool with the right parameters.

**Never** apply a workaround by patching the generated file directly and moving on.

---

## Rule 3 — No direct `.py` file edits

If a generated `.py` file has incorrect behavior (wrong class instantiation, missing imports, wrong method calls), the fix must come from the tool that generates it — not from editing the file.

If the tool itself has a bug that produces bad output, the correct actions are:
- Document what the generator is producing incorrectly.
- Work around it by restructuring the flow/API definition inputs so the generator produces correct output.
- Report the generator bug separately.

---

## Rule 4 — Validate before running

Before executing any flow or test suite, run `validate_flow` or `validate_files` to catch structural errors early.

---

## Rule 5 — Follow the existing patterns in the project

Before creating a new flow or API, check an existing working equivalent using `get_flow` or `get_api_definition`. Match the same structure for:
- `context_export` / `response_export` field names
- `header_import` for injecting extracted values into subsequent steps
- `session_config` for shared headers like `RecaptchaToken`
- Data source format (`test_case` key, `expected_status`, `payload`)

This prevents generation issues caused by using unsupported field names or formats.

---

## Rule 6 — Destructive and uncovered write tools

The negotiation rules cover *creating/updating* flows, APIs, queries, test groups, load tests, and report-dashboard actions, and **every environment write** (`manage_environment_variables`, `create_environment`, `update_environment`, `delete_environment`, `set_environment_db_connection` — see `negotiation/env-vars.md`). Every **other** write, destructive, or execution MCP tool is governed by this rule:

- **Deletes / duplicates / restores:** `delete_flow`, `delete_api_definition`, `delete_test_group`, `duplicate_flow`, `duplicate_api`, `copy_test_group`, `restore_flow_backup`
- **Assets:** `upload_asset`, `manage_asset` (write actions)
- **Prerequisite templates:** `manage_prerequisite_template` (`create` / `update` / `duplicate` / `delete`) — shared project data every API run can name
- **Job history:** `clear_jobs` — irreversible; it erases the run records the report dashboard is built on
- **Direct executions of existing artifacts:** `execute_test_group`, `run_test_suite`, `run_multiple_suites`, `run_api`, `run_test_case` — real traffic against a real environment
- **Suite generation:** `generate_test_suite` — regenerates files the generator owns

Before calling any of these:

1. **Name the exact target back to the user and confirm intent first** — never guess or invent a name/id, and never batch multiple destructive actions into one confirmation.
2. **Check dependents before deleting.** `delete_query` fails and returns `used_in` when referenced; flows and suites may be referenced by test groups — surface what depends on the target instead of forcing or working around it.
3. **For direct executions**, confirm the environment (`environment_id` / `BASE_URL`) and background-vs-foreground before running; report the returned `job_id`s afterwards. Present environment choices per `ai-rules/reference/selection-format.md`.
4. **For `restore_flow_backup`**, state which backup (name/timestamp from `list_flow_backups`) will overwrite the current flow definition.
5. **For `run_api` with `prerequisites`** (ad-hoc auth chaining — read `qa-flow://schemas/prerequisites` first), the confirmation must name the whole chain, not just the target: which APIs/flows run first and in what order, and which response field lands in which header/body field. It is real traffic from every step, including the logins. The chain is never saved — say so, so the user does not expect it to apply to later group/CI runs. Two additions: with **`auto_inject`** the server derives mappings you did not write, so state that it is on and what it is expected to wire (and report `prerequisite_results.auto_inferred` afterwards — never present an auto-derived injection as something the user asked for); and a **flow prerequisite with declared inputs** needs its `inputs` values confirmed with the rest of the chain, since they are test data the user owns, not a detail to invent.
6. **For `run_api`, say how many test cases will run.** With no `test_case` (or the default `"default"`/`"all"`) the call runs **every** test case the API defines — negative and error cases included, each one real traffic. When the user asked for one case, name it in `test_case` exactly as `get_test_cases` reports it; an unknown name is rejected with the valid list rather than running nothing.
7. **For `run_api` with a `prerequisite_template`**, a name is not a confirmation. Read the template with `get_prerequisite_template` and confirm its **expanded** contents — every step in order, every mapping, and the names of any input values it has stored (never their values) — exactly as for an inline chain, plus anything passed alongside it, which runs *after* the template. Say that the template is shared project data: it may have been edited by someone else since it was made — over MCP as well, via `manage_prerequisite_template` (item 9) — and a stored credential in it is whatever the project committed. If you pass **`prerequisite_template_test_cases`**, the confirmation must name which test case each step will run and which steps keep the case the template saved — a different case is different traffic, and the pick applies to that run only, never to the stored template. The same holds for **`flow_test_cases`** on a flow prerequisite (inline or inside a template): it re-points that flow's own steps, so name the flow, the step and the case(s) it will run instead of the ones the flow pins.
8. **For a data-driven `run_api`** (`dataset` / `dataset_mapping` / `dataset_execution` / `dataset_max_parallel`), every row is its own real request — and its own prerequisite chain or template run, so its own login. `test_case` must name ONE case. The confirmation names that case, the dataset source (asset / saved query / inline) with its row count from `preview_dataset` (and any `limit`/`filter`), each mapping row (`column → destination.path`), and sequential vs parallel with `dataset_max_parallel`. Omitting `dataset` while passing another `dataset_*` argument runs the binding already saved on the case — say so. Nothing passed here is saved on the case; persisting a binding is a `create_or_update_api` change under `negotiation/api.md`. Afterwards report `dataset_summary` (passed / failed / failing row ids), not only the overall outcome.
9. **For `manage_prerequisite_template` writes**, a template is **shared project data** in the project's committed `data/prerequisite_templates.json` — not a private scratch setting. Before `create` or `update`, confirm the template name and the whole chain being written: every step in order, every mapping (source field → destination header/body, with its prefix), and `carry_cookies`. Never store an input value the user did not explicitly ask to store — a password or OTP saved into a template is committed to the repo and reused by everyone who names it. `update` **replaces** the parts you pass (`steps` and `injections` are whole-list replacements, not merges), so read the template with `get_prerequisite_template` first and send the complete list. `delete` is destructive and cannot be undone from here: name the template, say that any 🔑 dialog setup referencing it will show a "missing template" row, and confirm before calling.
10. **Report the real result** — deleted/skipped counts, file paths, job ids — and never auto-chain another action.

---

## Rule 7 — Long negotiations and context loss

Negotiations can span dozens of turns and may outlive the assistant's context window (summarization / compaction — this applies to any assistant, not just one product).

- If you can no longer see the **full text of the governing rule file**, re-read it before continuing — do not proceed from memory of it.
- Keep a running **ledger of negotiated answers** (step-by-step decisions made so far). Restate it briefly every few questions, and always restate it before the final plan summary. If earlier answers have scrolled out of context, re-state what you still know and ask the user to confirm the ledger rather than re-asking everything.
