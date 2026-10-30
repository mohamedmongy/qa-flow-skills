# QA Flow — Global Values Rule

## Scope

Applies to **any AI assistant** whenever it is about to write the Global Values store — `manage_global_value` (create / update / delete / set_value / refresh / clear) or `set_api_global_injections` — and whenever another rule decides to reference a global value (`{{global.KEY}}`) in an artifact. Reading (`list_global_values`, `get_global_value` without `reveal`) is not gated.

Environment variables stay governed by `negotiation/env-vars.md`; secrets handling there applies here too (§4).

---

## 1. What a Global Value Is

A named value shared by every API, query and flow, stored in `data/global_values.json` (gitignored):

| Field | Meaning |
|---|---|
| `name` | `[A-Za-z_][A-Za-z0-9_]*` — referenced as `{{global.NAME}}` |
| `producer` | `{template, source_step?, path?}` — a saved **prerequisite template** run lazily when the value is missing or expired; `source_step`/`path` default to the template's first mapping onto the target. `null` = a value set by hand |
| `ttl_seconds` | How long a produced (or hand-set) value stays fresh. Produced values default to 1800; a hand-set value with no TTL never expires |
| `inject` | `{header, prefix, apis}` — the header every **listed API** receives automatically (inside flows too). Opt-in per API only |
| `sensitive` | Default `true` — values are masked everywhere unless revealed |

**One namespace, one cached value per key, for every environment.** A token produced against one environment is reused by a run against another until it expires. Before switching environment, propose `clear` (with no name) — never assume it was done.

A producer that fails **fails the run** that needed the value. A producer can't use `{{global.*}}` values that aren't cached yet.

**Header precedence** (later wins): the API's saved headers → the global injection (replaces a stale saved bearer) → the flow step's `header_import` → a run's prerequisite chain (`run_api` with `prerequisites` / `prerequisite_template`). A flow that logs in as its own user keeps sending that user's token even when the API is opted in.

**Flows can write values** with a step's `global_export` — `{"<name the step exports>": "<global key>"}` (the name must be in its `context_export` / `response_export`). A key that does not exist yet is **created on the fly** (sensitive, TTL 1800 s, no producer, no injection — the run log says so); an existing key keeps its settings and only its value is replaced. The value is cached for the key's TTL (or 1800 s for a key with a producer, else until replaced) and replaces whatever was cached — last writer wins, across environments too. A null export publishes nothing.

---

## 2. Reference Syntax

| Where | Form |
|---|---|
| API test-case `payload` / `params`, flow step `payload` / `params` / `query_params`, query step inputs, sub-flow inputs, flow input values, `ui` step scripts | `{{global.KEY}}` |
| Saved query parameter values — `run_saved_query`, ▶ Run on Manage Queries, `run_query_with_data` fixed parameters, a query dataset's `params`; the SQL text / Mongo filter of `test_query` | `{{global.KEY}}` |
| Flow `header_import[].variable` | `global.KEY` |
| Flow step writing a value | `"global_export": {"accessToken": "access_token"}` |
| Inside other `{{context.*}}` lookups | `{{context.global.KEY}}` (same value) |

`{{global.*}}` is **never resolved at save time** — it is safe in a saved API definition.

**Prefer injection over editing.** If what you need is "this API sends the shared token", opt the API in with `set_api_global_injections` instead of rewriting its headers.

---

## 3. Negotiation (standalone writes)

The first response is **one question**, with no tool calls before it. The only exception is `list_global_values`, which may run first to check whether the name is free or the key exists.

Ask in this order, one topic per message (`reference/selection-format.md`):

1. **Key**: create a new key or change an existing one. For a new key, propose a name, run the availability check, and suggest an alternative if it collides.
2. **Source**: a producer (pick a prerequisite template from `list_prerequisite_templates`, and the path only if the default mapping is wrong), or a value set by hand (§4).
3. **Settings**: ask about `ttl_seconds`, `sensitive` and the inject rule's header and prefix together as one question.
4. **Opt-ins** (only if there is an inject rule): which APIs receive the header, as one multi-select question.

**Confirm** with one summary: the name, the producer or the masked value, the TTL, whether it's sensitive, the inject rule, and the APIs.

- `delete`: list every artifact that references `{{global.KEY}}` or opts in to it, and get an explicit yes first.
- `refresh`: runs the producer for real, so it needs a yes.
- `clear`: needs a yes.
- `reveal=true`: only when the user asks to see the value, and never echo the value back in a summary.

Never run anything afterwards to "test" the key unless the user asks. A refresh is a run.

---

## 4. Secrets

A value set by hand that is a token, password or key follows `negotiation/env-vars.md` §5:

- Collect it in the conversation, never echo it, and keep `sensitive: true`.
- A producer is preferred over a hand-set token, because it renews itself.

---

**Adding `global_export` to a flow** is a flow edit: it follows `negotiation/flow.md`. Check the key name against `list_global_values` in the same question — a misspelt key is silently created as a new one, so reuse an existing name unless the user wants a new key.

## 5. Inside Other Workflows

When the flow, API or plan rules would hardcode an auth header, a login step or a prerequisite for a target API, and a global value with a matching inject rule exists, offer it as one option in the question that rule already asks. Options:

- opt the API in to the global value,
- reference `{{global.KEY}}`,
- or keep the in-flow login.

The calling workflow's single confirmation must list any `set_api_global_injections` change it makes.
