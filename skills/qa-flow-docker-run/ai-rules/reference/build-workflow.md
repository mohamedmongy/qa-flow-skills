# QA Flow — Build Workflow and Common Mistakes

---

## Build Sequence

Always follow this order. Skipping steps causes silent failures or runtime errors.

1. **`list_api_definitions`** — confirm which APIs exist
2. **`get_test_cases(api_name)`** for every API that will be used as a step — verify exact test case names
3. **`list_queries`** — if any steps use `query_method`, confirm the query exists; run `save_query` if not
4. **`list_flows`** — check if a similar flow exists; call `get_flow` on it for reference patterns only — never copy blindly
5. **Negotiate with the user** (see `ai-rules/negotiation/flow.md`) — never skip this step
6. **`validate_flow(flow_name, steps)`** — fix structural errors before saving
7. **`create_flow`** or **`update_flow`** — only after explicit user confirmation
8. **`generate_test_suite(flow_name)`** — regenerates the `.py` suite file if needed
9. **`run_test_suite(suite="code_generator/TestCases/test_<flow_name>.py", background=true, environment_id=...)`** — always `background=true` for flows
10. **`get_run_status(job_id)`** — poll until complete

**For `update_flow`:** always call `get_flow` first, modify the full steps array, then send the complete definition. The tool replaces the entire steps array — partial updates lose existing steps.

---

## Common Mistakes and Fixes

| Mistake | Why It Breaks | Fix |
|---|---|---|
| `"type": "api"` or `"api_name": "..."` | Unknown step type / missing `api` field | Use `"type": "api_call"` and `"api": "..."` |
| `context_export` as a dict `{"key": "path"}` | Generator expects a list | Use `"context_export": ["key"]` |
| `"test_case": "my_case"` when it doesn't exist | Step runs but silently does nothing | Call `get_test_cases` first to verify exact name |
| `"assertions": [...]` on a step | Field is not read by the generator | Put assertions inside the API's test case definition |
| `"headers": {...}` on a step | Field is not read by the generator | Use `header_import` for dynamic headers |
| `{{step_1.token}}` in a payload | Wrong template prefix — not resolved | Use `{{context.step_1.token}}` |
| `"variable": "context.step_1.token"` in header_import | Double prefix causes lookup failure | Use short form: `"variable": "step_1.token"` |
| Bare field name in DB condition `"field": "status"` | DB results are wrapped under `result` | Use `"field": "result.status"` |
| `"operator": "=="` (or `!=`, `is_not_empty`) in a `wait_until` visual condition | `wait_until` uses `ConditionEvaluator`, which raises `ValueError` on those | Use `eq`/`ne`/`gt`/`gte`/`lt`/`lte`/`contains`/`not_contains` for `wait_until` |
| `"operator": "eq"` (or `gt`, `lt`) in a `conditional` step | `conditional` uses a different evaluator that only knows symbolic operators | Use `==`/`!=`/`>`/`<`/`contains`/`is_empty`/`is_not_empty` for `conditional` |
| Referencing a future step in `context_import` | ContextManager validation fails at runtime | Only reference steps that execute before the current one |
| `query_method` without saving the query first | Method not found error at runtime | Call `save_query` first, confirm with `list_queries` |
| `"use_shared_session": false` with `session_config` defined | session_config headers are not applied | Set `"use_shared_session": true` |
| `"test_cases": "default"` (string instead of array) | Format mismatch — not processed as multiple cases | Use `"test_cases": ["default"]` |
| Hardcoding a Bearer token in a step | Token won't rotate, breaks other environments | Use `header_import` with `"variable": "step_N.accessToken"` and `"prefix": "Bearer "` |
| Calling `create_flow` without `validate_flow` first | Structural errors not caught until runtime | Always validate before saving |
| `update_flow` with only new/changed steps | Generator replaces the entire steps array | `get_flow` first → modify → send complete steps array |
| `suite="test_x.py"` path | Suite file not found | Use full path: `"suite": "code_generator/TestCases/test_x.py"` |
| Running with `background=false` for a multi-step flow | Call blocks until completion — can time out | Use `background=true` and poll `get_run_status` |

---

## Quick Reference — Step Types

| Situation | Use |
|---|---|
| Call an existing API once | `api_call` |
| Run multiple test cases on one API in sequence | `api_call` with `test_cases` array |
| Poll an API until a condition is met | `wait_until` with `api` + `test_case` |
| Poll a DB until a record appears or changes | `wait_until` with `query_method` |
| Fetch data from DB for use in later steps | `query` |
| Reuse another flow as a building block | `flow` |
| Branch logic based on a context value | `conditional` |
| Add a fixed time delay | `waitBefore` / `waitAfter` on any step, or standalone `wait` |
| Save a response file to Assets Manager | `file_downloads_code` on `api_call` or `wait_until` |
