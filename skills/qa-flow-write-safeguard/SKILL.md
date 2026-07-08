---
name: qa-flow-write-safeguard
description: MUST be used whenever the user asks for any QA Flow MCP write, destructive, or execution action NOT covered by a negotiation skill — deleting/duplicating/restoring flows, APIs, or test groups (delete_flow/duplicate_flow/restore_flow_backup/delete_api_definition/duplicate_api/delete_test_group/copy_test_group), environment writes (create/update/delete_environment, manage_environment_variables, set_environment_db_connection), assets (upload_asset/manage_asset), clearing job history (clear_jobs), generating suites (generate_test_suite), or directly running an existing test group/suite/flow/API (execute_test_group/run_test_suite/run_multiple_suites/run_api/run_test_case). Ensures the MCP safeguard rules are read, the exact target and dependents are confirmed, and the environment is confirmed before any real execution. Does NOT apply to read-only tools (list_*/get_*/health_check) or to create/update actions covered by the negotiation skills.
---

# QA Flow — Write Safeguard (catch-all)

Any QA Flow MCP **write, destructive, or execution** action that is not governed by one of the negotiation skills (flow/API, test group, query, load test, report dashboard) is governed by the MCP safeguard rules — especially **Rule 6** (destructive and uncovered write tools). This skill exists only to make sure they are loaded and followed. **On any conflict between this summary and the rule file, the rule file wins.**

## Do this first — read the authoritative rule

- **All safeguard rules (Rules 1–7)** → [ai-rules/safeguard.md](ai-rules/safeguard.md)

If the request is actually a create/update of a flow, API, query, test group, load test, or report-dashboard action, stop — that is the matching negotiation skill's territory, not this one's.

## The non-negotiable constraints (full detail is in the file above)

1. **Confirm the exact target and intent first.** Name the target back to the user before deleting, overwriting, restoring, or running it — never guess or invent names/ids, and never batch several destructive actions under one confirmation.
2. **Check dependents before deleting.** Flows/suites/queries may be referenced by test groups or other flows — surface what depends on the target (`used_in`, group items) instead of forcing or working around it.
3. **Direct executions are real traffic.** For `execute_test_group` / `run_test_suite` / `run_multiple_suites` / `run_api` / `run_test_case`, confirm the environment (`environment_id` / `BASE_URL`) and background-vs-foreground before running; report the `job_id`s after.
4. **`clear_jobs` is irreversible** — it erases the run history the report dashboard is built on. State that plainly before proceeding.
5. **`restore_flow_backup` overwrites the current flow** — state which backup (from `list_flow_backups`) will replace it.
6. **Never hand-edit generated files** (`.py`, `.json`, test suites) — the MCP tools own them (Rules 1–3).
7. **Report the real result** — deleted/skipped counts, file paths, job ids — and never auto-chain another action.

## Session resilience (any assistant)

If you can no longer see the safeguard file's full text (long conversation / compaction), re-read it before continuing (safeguard Rule 7). Present choices per [ai-rules/reference/selection-format.md](ai-rules/reference/selection-format.md).
