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

The negotiation rules cover *creating/updating* flows, APIs, queries, test groups, load tests, and report-dashboard actions. Every **other** write, destructive, or execution MCP tool is governed by this rule:

- **Deletes / duplicates / restores:** `delete_flow`, `delete_api_definition`, `delete_test_group`, `duplicate_flow`, `duplicate_api`, `copy_test_group`, `restore_flow_backup`
- **Environments:** `create_environment`, `update_environment`, `delete_environment`, `manage_environment_variables`, `set_environment_db_connection`
- **Assets:** `upload_asset`, `manage_asset` (write actions)
- **Job history:** `clear_jobs` — irreversible; it erases the run records the report dashboard is built on
- **Direct executions of existing artifacts:** `execute_test_group`, `run_test_suite`, `run_multiple_suites`, `run_api`, `run_test_case` — real traffic against a real environment
- **Suite generation:** `generate_test_suite` — regenerates files the generator owns

Before calling any of these:

1. **Name the exact target back to the user and confirm intent first** — never guess or invent a name/id, and never batch multiple destructive actions into one confirmation.
2. **Check dependents before deleting.** `delete_query` fails and returns `used_in` when referenced; flows and suites may be referenced by test groups — surface what depends on the target instead of forcing or working around it.
3. **For direct executions**, confirm the environment (`environment_id` / `BASE_URL`) and background-vs-foreground before running; report the returned `job_id`s afterwards. Present environment choices per `ai-rules/reference/selection-format.md`.
4. **For `restore_flow_backup`**, state which backup (name/timestamp from `list_flow_backups`) will overwrite the current flow definition.
5. **Report the real result** — deleted/skipped counts, file paths, job ids — and never auto-chain another action.

---

## Rule 7 — Long negotiations and context loss

Negotiations can span dozens of turns and may outlive the assistant's context window (summarization / compaction — this applies to any assistant, not just one product).

- If you can no longer see the **full text of the governing rule file**, re-read it before continuing — do not proceed from memory of it.
- Keep a running **ledger of negotiated answers** (step-by-step decisions made so far). Restate it briefly every few questions, and always restate it before the final plan summary. If earlier answers have scrolled out of context, re-state what you still know and ask the user to confirm the ledger rather than re-asking everything.
