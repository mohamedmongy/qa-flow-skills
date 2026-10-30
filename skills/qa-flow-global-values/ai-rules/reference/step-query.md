# QA Flow — Step Type: `query`

Executes a saved DB query (Mongo or SQL) and optionally exports results to flow context.

## Schema

```json
{
  "type": "query",
  "name": "step_4",
  "query_method": "get_user_subscriptions",
  "query_params": {
    "client_id": "{{context.env.CLIENTID}}",
    "feature_id": "{{context.flow_input.feature_id}}",
    "status_id": "{{context.env.STATUSID}}"
  },
  "context_export": ["result.bundle_id", "result.quota_value"],
  "context_import": []
}
```

## Field Reference

| Field | Required | Notes |
|---|---|---|
| `query_method` | **yes** | Must match a saved query method name exactly — call `list_queries` to verify |
| `query_params` | yes | Key-value map of parameters; supports `{{context.*}}` template variables |
| `context_export` | no | Fields from the query result to export. DB results are accessed as `result.<field>` |
| `context_import` | no | Declare dependencies on upstream context values |

## Prerequisite: The Query Must Exist First

The query method must be saved before the flow is created. If it doesn't exist:

1. Call `save_query` to create it
2. Call `run_saved_query` to smoke-test it
3. Only then reference it in the flow step

Call `list_queries` to see what's available.

## Accessing Query Results

Query results are automatically wrapped under `result` in the context. To export a field:

```json
"context_export": ["result.status", "result.bundle_id"]
```

After export, downstream steps reference it as `step_4.result.bundle_id`.

## Array Indexing in context_export

When the query returns a list, use array index notation:

```json
"context_export": ["result[0].record_guid", "result[0].bundle_id"]
```

Exported as `step_4.result[0].record_guid`.

## Type Conversion

The runner automatically converts string parameter values to the type declared in the query method's Python signature:
- Annotated `int` → converted via `int(value)`
- Annotated `float` → converted via `float(value)`
- Annotated `bool` → `true/1/yes` → `True`
- Empty string for an optional parameter → converted to `None`

## Template Variables in query_params

All `{{context.*}}` template syntax is supported:

```json
"query_params": {
  "client_id":  "{{context.env.CLIENTID}}",
  "feature_id": "{{context.flow_input.feature_id}}",
  "user_id":    "{{context.login_step.userId}}"
}
```
