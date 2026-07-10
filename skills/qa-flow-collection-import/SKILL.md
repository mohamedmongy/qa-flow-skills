---
name: qa-flow-collection-import
description: MUST be used whenever the user asks to bulk-import a request collection into QA Flow API definitions — a Postman collection (`.postman_collection.json`) or a Swagger/OpenAPI spec — via the qa-flow MCP server. Triggers on natural-language requests like "import this Postman collection", "add this Swagger/OpenAPI spec as APIs", "bulk-import these endpoints", "convert this collection into QA Flow APIs", or "turn this folder of requests into APIs". Enforces the project's negotiate-before-you-build rules for bulk imports — parse and show the inventory first, negotiate the global conventions ONCE (variable/URL mapping, auth, naming, test-case policy, batching) one topic per message, then confirm and build ONE folder/tag at a time (single list_api_definitions name check per batch, no other MCP tools before negotiation), never auto-run what was built. Does NOT apply to a single cURL / one endpoint (use qa-flow-flow-negotiation) or to information-only requests (list folders, show the collection).
---

# QA Flow — Collection Import (Postman / Swagger) Negotiation

Bulk-importing a Postman collection or Swagger/OpenAPI spec into QA Flow **API definitions** is governed by strict negotiation rules. These rules are the source of truth — this skill exists only to make sure they are loaded and followed. **On any conflict between this summary and the rule files, the rule files win**; keep this summary in sync when the rules change.

## Do this first — read the authoritative rules

Before doing anything else, Read these and follow them exactly:

- **Bulk collection import** → [ai-rules/negotiation/collection-import.md](ai-rules/negotiation/collection-import.md) — the orchestration rule for this workflow.
- **Each API it builds** → [ai-rules/negotiation/api.md](ai-rules/negotiation/api.md) — every API produced still conforms to this (naming, test-case structure, Static Values → Env Vars).
- **Any QA Flow MCP write** → [ai-rules/safeguard.md](ai-rules/safeguard.md) — env-var creation and per-batch confirmation obey it.

Present every choice per [ai-rules/reference/selection-format.md](ai-rules/reference/selection-format.md).

## The non-negotiable constraints (full detail is in the files above)

1. **Source first.** If the request doesn't point to a collection/spec (path, attachment, or pasted JSON/YAML), the FIRST response asks for it — zero tool calls. Skip the ask when it was already provided.
2. **Parse, then show the inventory.** Reading the collection file is a local read (not an MCP call) and is allowed. Present a read-only summary — folders with request counts, the detected `{{variables}}` (flag typo-duplicates) — before any decision.
3. **Negotiate the global conventions ONCE, one topic per message** — (1) variable/URL mapping, (2) auth, (3) naming, (4) test-case policy, (5) batching. Related low-priority settings are grouped into a single preset-menu question. These answers form a **convention ledger** applied to every API; restate it if the conversation is long enough that it scrolls out of view (safeguard Rule 7). Never bundle unrelated topics; wait for each answer.
4. **Build ONE folder/tag per batch, each confirmed.** The only MCP tool allowed before a batch is built is a single `list_api_definitions` name-availability check for that batch. A collision always resolves to a NEW name — never overwrite an existing API. Present the batch table (request → api_name, method, endpoint, promoted vars, test case) and get explicit confirmation before calling `create_or_update_api`.
5. **Create missing env vars first**, then build. The dashboard rejects a curl whose `{{env.VAR}}` isn't in the environment — add it via `manage_environment_variables` (per safeguard) before the APIs that reference it.
6. **After a batch, report and never auto-run.** Ask before running anything. Re-confirm the next folder at the batch step before building it.

If you find yourself about to call `create_or_update_api` (or `manage_environment_variables`) before the conventions and the current batch are confirmed, stop and ask the next negotiation question instead.

## When this skill does NOT apply

- A single cURL or one endpoint → use **qa-flow-flow-negotiation** (the `api.md` rule) directly.
- Information-only requests ("list the folders", "show me the collection") → just answer.

## Session resilience (any assistant)

- Long imports can outlive the context window. If you can no longer see the rule file's full text or the convention ledger, **re-read `collection-import.md`** and restate the ledger + the folders already built before asking the next question (safeguard Rule 7).
- If the client has a native single-question picker (e.g. AskUserQuestion), prefer it for every selection — small fixed sets as options; long folder/variable lists via the hybrid pattern (full table in the message + top candidates as picker options) — still one question per message.
