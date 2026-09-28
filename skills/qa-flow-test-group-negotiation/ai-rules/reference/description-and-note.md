# QA Flow — Description and Note (shared)

This is the single authoritative copy of how an assistant handles an artifact's **description** and **note** while negotiating it. It is referenced by `negotiation/api.md`, `negotiation/query.md`, `negotiation/flow.md`, and `negotiation/test-group.md`.

---

## Two different fields

| | Description | Note |
|---|---|---|
| **What** | What the artifact is *for* — one sentence | The user's checklist for its *results* |
| **Read by** | People scanning lists, and assistants reading `list_*` results (only the **first line**, ≤160 chars, is listed) | The person reading a run result — shown with every run of the artifact, alone or inside a flow / test group, and in the Reports tab, the HTML / PDF / CSV / Excel exports and the CI report channels |
| **Who writes it** | The assistant **drafts** one; the user keeps, edits, or skips it | **Only the user.** Never draft or invent a note |
| **Limit** | 1000 chars | 2000 chars |
| **Carried by** | APIs, queries, flows, test groups (and assets, prerequisite templates, environments) | APIs, queries, flows, test groups |

Both are optional. An update that leaves either out keeps the stored value; `""` clears it.

---

## During negotiation — folded into the name question

The description and note are **not** extra questions. They ride along in the message that settles the name (the flow and test-group rules already propose a description there):

```
I suggest naming this API `get_user_info` (available).
Description: "Fetches the signed-in subscriber's profile (name, email, card state)".
(1) keep both  (2) change the name  (3) change the description  (4) no description
Optional: a note — a checklist you'll see with every run result (e.g. "- status is ACTIVE").
```

**Drafting the description:**

- One sentence, in plain words, saying what the artifact does or returns — not how (no endpoint paths or SQL restated).
- The **first line must stand alone**: it is all a `list_*` result shows.
- Derive it from what the user said and the source (the cURL's method + resource, the query's purpose, the flow's goal, the group's contents). If the purpose is genuinely unclear, draft nothing and ask for one in the same question.
- The user's answer wins verbatim. "No description" is a valid answer — never re-ask.

**The note:** mention it **once**, in that same message, as optional. Record exactly what the user writes. If they don't give one, there is no note — never fill one in from the assertions or scenarios.

---

## In the plan and the build

- The Step 3 summary lists **Description** and **Note** (or "none") next to the name.
- The build passes them to the create / update tool:
  - `create_or_update_api(..., description=…, note=…)`
  - `save_query(..., description=…, note=…)` for any kind; `update_query(query_name, kind='meta', description=…, note=…)` changes only these two on an existing query
  - `create_flow` / `update_flow(..., description=…, note=…)`
  - `create_test_group(..., description=…, note=…)`; on `update_test_group` they are fields of the group object
  - an asset: `manage_asset(filename, 'set_description', description=…)`
- Leave a field out when the user gave none — sending `""` on an update clears what is stored.
