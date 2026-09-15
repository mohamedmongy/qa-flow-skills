# QA Flow — File Downloads in Flow Steps

After a successful `api_call` or `wait_until` step, use `file_downloads_code` to save a file from the response to the Assets Manager.

---

## Schema

Add `file_downloads_code` as a Python string on any `api_call` or `wait_until` step:

```json
{
  "type": "api_call",
  "name": "export_report",
  "api": "export_csv_api",
  "test_case": "default",
  "file_downloads_code": "save_file('_raw', 'report.csv', key='csv_report', mime_type='text/csv')"
}
```

---

## The save_file Helper

```python
save_file(response_path, file_name, key='', mime_type='')
```

| Parameter | Required | Notes |
|---|---|---|
| `response_path` | yes | Dot-notation path into the response body, or `'_raw'` for raw bytes |
| `file_name` | yes | Filename to write in `data/assets/` |
| `key` | no | Logical key for referencing this file in later steps |
| `mime_type` | no | MIME type override stored in the Assets Manager index |

`save_file` can be called multiple times in one snippet to save multiple files.

---

## response_path Values

| Value | What It Downloads |
|---|---|
| `'_raw'` | The entire raw response body (bytes) |
| `'response.data.exportUrl'` | Fetches a URL found at that dot-path in the JSON response |
| `'response.data.fileContent'` | Saves the string/bytes value at that dot-path |

---

## Available Variables in the Snippet

| Variable | Type | Notes |
|---|---|---|
| `save_file(...)` | function | The download helper |
| `response` | ParsedResponseTree / dict | Parsed response body from the last test case |
| `response_raw` | bytes | Raw response bytes |
| `status_code` | int | HTTP status code |
| `context` | dict | Current flow context (read-only recommended) |

---

## Auto-Export to Context After Saving

After `save_file` completes, the generator automatically exports the file into context under several keys:

| Context key | Value |
|---|---|
| `<step_name>.file_name` | The actual filename (always) |
| `file_name` | The actual filename, unprefixed (always) |
| `<step_name>.<key>` | The actual filename (when `key` was provided) |
| `assets.<file_name>` | The actual filename (always — accessible as `{{assets.<file_name>}}`) |
| `assets.<key>` | The actual filename (when `key` was provided — accessible as `{{assets.<key>}}`) |

---

## Using a Downloaded File in a Later Step

Once saved with a key, reference the file in a subsequent upload step:

```json
{
  "type": "api_call",
  "name": "upload_report",
  "api": "upload_file_api",
  "test_case": "default",
  "payload": {
    "file": "{{assets.csv_report}}"
  }
}
```

---

## Multi-File Example

```python
# Save raw body as CSV
save_file('_raw', 'export.csv', key='csv_export', mime_type='text/csv')

# Save an Excel URL from the response JSON
save_file(
    'response.data.xlsxUrl',
    'export.xlsx',
    key='xlsx_export',
    mime_type='application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
)
```

---

## Notes

- `file_downloads_code` only runs after a **successful** step (all test cases passed)
- The file is written to `data/assets/` and registered in `data/assets/index.json`
- If the response_path is a URL string (starts with `http://` or `https://`), the runner fetches the URL and saves its content
- If the file already exists, it is overwritten and the previous version is recorded in the index history
