# QA Flow — Flow JSON Structure

A flow is a JSON object with these top-level fields:

```json
{
  "flow_name": "my_flow",
  "description": "What this flow tests",
  "use_shared_session": false,
  "session_config": null,
  "inputs": [],
  "steps": []
}
```

| Field | Required | Notes |
|---|---|---|
| `flow_name` | yes | `snake_case`; becomes the test file name |
| `description` | no | Short description shown in the dashboard |
| `use_shared_session` | no | Set `true` when `session_config` is defined |
| `session_config` | no | Object with a `headers` map — injected into every step |
| `inputs` | no | Array of runtime input declarations |
| `steps` | yes | Ordered array of step objects |
| `expected_flow_execution_time` | no | Timeout hint in seconds |

Every step inside `steps` **must** have `"name"` and `"type"`. The step name becomes the context prefix for all values exported by that step (e.g., step named `get_csrf` produces context key `get_csrf.csrfToken`).

## Related Files

- Step types → `ai-rules/reference/step-api-call.md`, `ai-rules/reference/step-query.md`, `ai-rules/reference/step-wait-until.md`, `ai-rules/reference/step-conditional-and-flow.md`
- Context wiring, shared headers (`session_config`), and env vars → `ai-rules/reference/context-wiring.md`
- File downloads → `ai-rules/reference/file-downloads.md`
- API / test-case selection → `ai-rules/reference/api-and-testcase-selection.md`
- Build workflow → `ai-rules/reference/build-workflow.md`
