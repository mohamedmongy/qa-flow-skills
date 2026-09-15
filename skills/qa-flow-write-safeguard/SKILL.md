---
name: qa-flow-write-safeguard
description: MUST be used whenever the user asks for any QA Flow MCP write, destructive, or execution action NOT covered by a negotiation skill — deleting/duplicating/restoring flows, APIs, or test groups (delete_flow/duplicate_flow/restore_flow_backup/delete_api_definition/duplicate_api/delete_test_group/copy_test_group), assets (upload_asset/manage_asset), prerequisite templates (manage_prerequisite_template), clearing job history (clear_jobs), generating suites (generate_test_suite), or directly running an existing test group/suite/flow/API (execute_test_group/run_test_suite/run_multiple_suites/run_api/run_test_case) — including running an API behind an ad-hoc prerequisite chain or a saved prerequisite template ("run X with the login template", run_api with prerequisites/auto_inject/prerequisite_template). Ensures the MCP safeguard rules are read, the exact target and dependents are confirmed, the whole prerequisite chain is expanded and confirmed, and the environment is confirmed before any real execution. Does NOT apply to read-only tools (list_*/get_*/health_check), to environment writes (manage_environment_variables, create/update/delete_environment, set_environment_db_connection — see qa-flow-env-vars), or to create/update actions covered by the negotiation skills.
---

# QA Flow — Write Safeguard (catch-all)

Any QA Flow MCP **write, destructive, or execution** action that is not governed by one of the negotiation skills (flow/API, test group, query, load test, report dashboard) is governed by the MCP safeguard rules — especially **Rule 6** (destructive and uncovered write tools). This skill exists only to make sure they are loaded and followed. **On any conflict between this summary and the rule file, the rule file wins.**

## Do this first — read the authoritative rule

- **All safeguard rules (Rules 1–7)** → [ai-rules/safeguard.md](ai-rules/safeguard.md)

If the request is actually a create/update of a flow, API, query, test group, load test, or report-dashboard action, stop — that is the matching negotiation skill's territory, not this one's. Environment variables, environments, and environment DB connections belong to **qa-flow-env-vars** ([ai-rules/negotiation/env-vars.md](ai-rules/negotiation/env-vars.md)).

## The non-negotiable constraints (full detail is in the file above)

1. **Confirm the exact target and intent first.** Name the target back to the user before deleting, overwriting, restoring, or running it — never guess or invent names/ids, and never batch several destructive actions under one confirmation.
2. **Check dependents before deleting.** Flows/suites/queries may be referenced by test groups or other flows — surface what depends on the target (`used_in`, group items) instead of forcing or working around it.
3. **Direct executions are real traffic.** For `execute_test_group` / `run_test_suite` / `run_multiple_suites` / `run_api` / `run_test_case`, confirm the environment (`environment_id` / `BASE_URL`) and background-vs-foreground before running; report the `job_id`s after.
4. **A prerequisite chain is confirmed whole, expanded.** `run_api` with `prerequisites` runs real logins before the target: name every step in order and every response field → header/body mapping, say `auto_inject` is on and what it will wire, and confirm a flow prerequisite's `inputs` values. A `prerequisite_template` is a name, not a confirmation — read it with `get_prerequisite_template` and confirm its expanded steps, mappings and the *names* of any stored inputs (never their values), plus anything passed alongside it, which runs after it. Templates are shared project data (someone else may have changed one) and are read-only over MCP; `prerequisite_template_test_cases` changes which case a step inside it runs — name each pick in the confirmation, for that run only.
5. **`clear_jobs` is irreversible** — it erases the run history the report dashboard is built on. State that plainly before proceeding.
6. **`restore_flow_backup` overwrites the current flow** — state which backup (from `list_flow_backups`) will replace it.
7. **Never hand-edit generated files** (`.py`, `.json`, test suites) — the MCP tools own them (Rules 1–3).
8. **Report the real result** — deleted/skipped counts, file paths, job ids — and never auto-chain another action.

## Session resilience (any assistant)

If you can no longer see the safeguard file's full text (long conversation / compaction), re-read it before continuing (safeguard Rule 7). Present choices per [ai-rules/reference/selection-format.md](ai-rules/reference/selection-format.md).
