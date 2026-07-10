---
name: qa-flow-query-negotiation
description: MUST be used whenever the user asks to create, update, duplicate, or delete a QA Flow saved database query (save_query/update_query/duplicate_query/delete_query) — SQL (PostgreSQL) or NoSQL (MongoDB) — via the qa-flow MCP server. Includes natural-language requests like "save this query", "make a DB check/lookup for flows", or "parameterize this SQL". Enforces the project's negotiate-before-you-build rules — one topic per message (related low-priority settings like all parameters + return type grouped into one question), the statement/kind first when the request lacks it, a single list_queries name-availability check before the name question and no other MCP tools before negotiation, get explicit confirmation before building. Does NOT apply to information-only requests (list queries, show query X).
---

# QA Flow — Database Query Negotiation

Creating or updating QA Flow **saved database queries** (SQL or NoSQL) in this repo is governed by strict negotiation rules. These rules are the source of truth — this skill exists only to make sure they are loaded and followed. **On any conflict between this summary and the rule files, the rule files win**; keep this summary in sync when the rules change.

## Do this first — read the authoritative rule

Before doing anything else, Read the matching file and follow it exactly:

- **Query** (`save_query` / `update_query` / `duplicate_query` / `delete_query`) → [ai-rules/negotiation/query.md](ai-rules/negotiation/query.md)
- **Any QA Flow MCP write** → also [ai-rules/safeguard.md](ai-rules/safeguard.md)

The reference files in [ai-rules/reference/](ai-rules/reference/) (step-type guides, context wiring, selection format) are read on demand exactly when the negotiation rule tells you to.

## The non-negotiable constraints (full detail is in the files above)

1. **The query first, then one pre-negotiation tool call — the name-availability check.** If the request doesn't say what the query should do (pasted SQL/Mongo statement or a described lookup) and which kind it is (sql/mongo/definition), the FIRST response asks exactly that — zero tool calls; skip whichever half was already stated. Once purpose and kind are known, derive a candidate method name, call `list_queries` once to check it, and ask the name question — proposing the name if it's free, or suggesting 2–3 alternatives if it's taken (a collision always means a new name — never update the existing query as a fallback). Nothing else runs before negotiation — not `list_databases`, not `get_db_schema`, not `validate_sql`. "Gathering context first" beyond that single check is the forbidden pattern itself.
2. **One topic per message.** High-stakes decisions (name+kind, database, destructive intent) get their own question; related low-priority parameters are grouped — target+operation in one message, **all parameters + return type in one table-style message**. Never bundle unrelated concerns or a multi-step plan into one reply. Wait for each answer.
3. **Negotiate every concern** — kind (sql/mongo/definition), database, target, operation, query body, parameters, return type — even when a similar query already exists. Existing queries are reference only.
4. **Never hardcode static literals.** Dynamic/test values become parameters; environment/secret values become env vars (`{{env.VAR}}`). Ask where each literal should live.
5. **Flag destructive writes.** INSERT/UPDATE/DELETE/raw_sql (SQL) and Mongo `update` mutate real data — call it out and confirm intent + environment. `delete_query` fails when the query is still referenced (`used_in`) — surface the dependents, never force.
6. **Get explicit confirmation before building.** Present the plan for confirmation only *after* every concern has been negotiated one question at a time. Validate SQL (`validate_sql`) before `save_query` — but do **not** run the saved query as part of the build.
7. **After building, ask (as its own message) whether to smoke-test it** (`run_saved_query` / `test_query`) — never auto-run.

If you find yourself about to call a qa-flow query write tool and you have not yet completed negotiation, stop and ask the next negotiation question instead.

## Session resilience (any assistant)

- Long negotiations can outlive the context window. If you can no longer see the rule file's full text or the answers gathered so far, **re-read the rule file** and restate the negotiated-answers ledger before asking the next question (safeguard Rule 7).
- Present choices per [ai-rules/reference/selection-format.md](ai-rules/reference/selection-format.md). If the client has a native single-question picker (e.g. AskUserQuestion), prefer it for every selection — small fixed sets as options directly; long lists via the hybrid pattern (full text table in the message + top 3–4 candidates as picker options) — still one question per message.
