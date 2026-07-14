---
name: qa-flow-flow-import
description: MUST be used whenever the user asks to create a QA Flow flow from a steps document — a manual-test / test-steps file (`.txt`/`.md` or pasted steps) whose steps carry explicit technical bindings (HTTP endpoints + payloads or SQL + params per step, `{{env.*}}`/`{{context.*}}` wiring, wait/poll timeouts, subflow references, expected results) — via the qa-flow MCP server. Triggers on natural-language requests like "create/build this flow from these steps", "import this test steps document as a flow", "turn this manual test into a flow", or "one-shot this flow from the doc". Enforces parse-resolve-walk-the-steps: parse the document locally into an INTERNAL ledger (never displayed to the user), run ONE enumerated read-only resolution pass (list_flows/list_queries/list_api_definitions/get_test_cases/get_flow on subflows/get_environment), then confirm the steps with the user ONE per message (reuse what exists / create what's missing per its type, native picker preferred), and finish with ONE compact summary + ONE build confirmation authorizing the whole dependency chain (env vars → saved queries each smoke-tested → APIs → one create_flow → validate_flow); missing subflows block with options (user-described steps become their own NEW flow under the document's subflow name — never flattened into the parent as individual steps), collisions never overwrite, wiring integrity is enforced (no dangling `{{context.*}}` references — missing exports retro-proposed at the producer's turn; step renames propagate to all context prefixes), failures checkpoint, nothing is auto-run. Does NOT apply to a flow described only as a goal or informal step list without bindings (use qa-flow-flow-negotiation), to bulk-importing a Postman/Swagger collection (use qa-flow-collection-import), or to information-only requests (summarize the doc, list its steps).
---

# QA Flow — Flow Import (Steps Document)

Creating a flow from a **fully-specified steps document** is governed by strict import rules. These rules are the source of truth — this skill exists only to make sure they are loaded and followed. **On any conflict between this summary and the rule files, the rule files win**; keep this summary in sync when the rules change.

## Do this first — read the authoritative rules

Before doing anything else, Read these and follow them exactly:

- **The import workflow** → [ai-rules/negotiation/flow-import.md](ai-rules/negotiation/flow-import.md) — the orchestration rule for this workflow.
- **The document format & eligibility checklist** → [ai-rules/reference/steps-doc-format.md](ai-rules/reference/steps-doc-format.md) — step idioms → step types, and what "fully specified" means.
- **The flow it builds** → [ai-rules/negotiation/flow.md](ai-rules/negotiation/flow.md) — flow structure and step semantics still conform to it (and the `reference/` files it points to).
- **Each saved query / API it builds** → [ai-rules/negotiation/query.md](ai-rules/negotiation/query.md) / [ai-rules/negotiation/api.md](ai-rules/negotiation/api.md) — structural conformance.
- **Any QA Flow MCP write** → [ai-rules/safeguard.md](ai-rules/safeguard.md).

Present every choice per [ai-rules/reference/selection-format.md](ai-rules/reference/selection-format.md).

## The non-negotiable constraints (full detail is in the files above)

1. **Source first.** If the request doesn't point to a steps document (path, attachment, or pasted steps), the FIRST response asks for it — zero tool calls. Skip the ask when it was already provided.
2. **Eligibility gate.** The document must pass the fully-specified checklist in `steps-doc-format.md`. A natural-language tester table (actions + expected results, no bindings) is NOT this skill's territory — hand it to **qa-flow-flow-negotiation**. Partially-specified documents proceed with the unbound steps flagged and individually negotiated before confirmation.
3. **Parse locally into the INTERNAL ledger, then ONE read-only resolution pass** — exactly: `list_flows` (name check), `list_queries`, `list_api_definitions`, `get_test_cases` on matched APIs, `get_flow` on referenced subflows, `get_environment` (env-var existence). Nothing else runs before the final build confirmation — no writes, no `validate_*`, no runs.
4. **The ledger is agent state — never shown to the user.** It drives a **per-step confirmation loop**: one step per message (native picker preferred) — target exists → confirm reuse (recommended pick); missing query/API/test case → confirm creating it from the doc's bindings per its type; everything that step still needs (env-var values, placeholders, cadence defaults, static-literal placements) is asked inside its one message; shared entities confirm once. Never dump the ledger, a full plan, or several steps' decisions in one message. **Context-wiring integrity is enforced per the rule's *Context Wiring Integrity* section**: every consumed `{{context.step.field}}` must trace to a producing step's export (a dangling edge is retro-repaired at the producer's turn, never built), each step's question shows its resolved imports and its exports' consumers, and any step rename immediately rewrites all downstream `{{context.*}}` prefixes.
5. **Missing subflows block at their step's turn.** They are never guessed or synthesized — offer: import their own steps doc first / pick an existing flow (near-purpose candidates suggested) / build the user-described steps as a **NEW flow under the document's subflow name**. **The parent ALWAYS keeps its `flow` step as the document specifies — described steps (e.g. "csrf then login") are never flattened into the parent as individual steps** (inlining only on an explicit restructure request, confirmed as a deviation); the resolved subflow exports every context key the parent consumes. Collisions (same name, different content) always resolve to a NEW name — never overwrite. Matching content is reused (idempotent re-import).
6. **After the loop: ONE compact summary + ONE build confirmation** — reused vs to-create (counts + names, gathered env values with secrets masked), loop decisions, flow-level config, ending in the single "build it?" question. Never zero questions, never silent creation.
7. **Build in dependency order after the yes**: env vars (`manage_environment_variables`) → saved queries (`save_query`, each smoke-tested with `test_query`; a failure pauses the build at a reported ✅/⏸ checkpoint — no silent rollback, no auto-delete) → APIs (`create_or_update_api`, test-case additions are additive) → **one** `create_flow` → `validate_flow`.
8. **Never auto-run.** After building, ask — as its own message — whether to run the flow (flow.md Step 5).

If you find yourself about to call a write tool before the summary is confirmed, stop and repair the loop/summary instead.

## When this skill does NOT apply

- A flow described as a goal or informal steps without bindings → **qa-flow-flow-negotiation**.
- A Postman collection / Swagger spec of endpoints → **qa-flow-collection-import**.
- Information-only requests ("summarize this steps doc", "what would this create?") → parse locally and answer; no resolution pass, no negotiation.

## Session resilience (any assistant)

- Long imports can outlive the context window. If you can no longer see the rule file's full text or the internal ledger's configured state, **re-read `flow-import.md`** and restate the configured-steps checkpoint before continuing (safeguard Rule 7).
- If the client has a native single-question picker (e.g. AskUserQuestion), prefer it for every loop question, blocker/ambiguity choice, and the final confirmation — still one question per message.
