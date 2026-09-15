---
name: qa-flow-spec-plan
description: MUST be used whenever the user asks to turn a test/feature/E2E **spec document** into QA Flow automation — a spec describing journeys, acceptance criteria and the cURLs they call (e.g. `e2e-spec.md`, a Jira/QA spec, an acceptance-criteria doc), as opposed to pre-bound per-step instructions. Triggers on requests like "plan automation for this spec", "turn this spec into flows/APIs", "what would we automate from this spec", "read this spec and build the test plan", or "create the automation plan for MS-1234". Produces exactly ONE artifact — a **build plan** file at `test-plans/<slug>.build-plan.md` (prose + executable JSON blocks) — and **creates nothing in QA Flow**: no APIs, no queries, no flows, no groups, no env vars, no writes at all. Enforces parse → ONE read-only resolution pass (list_api_definitions/get_test_cases/list_queries/list_flows/list_test_groups/list_prerequisite_templates/get_environment) → ONE grouped question per section (all of an API's decisions — name, env vars, test cases, assertions, tags — in a single message, never one parameter at a time) → plan file → a spec-alignment check verifying every journey and acceptance criterion is accounted for. Existing APIs are reported as existing and reused; missing ones are planned from the spec's cURL with query string and body split correctly; auth needs resolve to an existing prerequisite template, a new one, or in-flow login wiring; SQL/NoSQL statements become saved queries; the journeys are packaged into flows by an explicit composition decision (one flow for the whole spec by default, subflows + a master flow only for a stated reason — never one flow per API call) with context wiring derived automatically; unautomatable journeys (client-side UI, spec-declared out-of-scope) are excluded with a documented reason, never invented around. Building the plan is the separate **qa-flow-plan-execute** skill. Does NOT apply to a fully-specified steps document with per-step endpoints/SQL/wiring (use qa-flow-flow-import), a Postman/Swagger collection (use qa-flow-collection-import), a single cURL or an informally-described goal (use qa-flow-flow-negotiation), or information-only requests ("summarize this spec").
---

# QA Flow — Spec Plan (Spec Document → Build Plan)

Turning a spec into automation is governed by strict rules. This skill exists only to make sure they are loaded and followed. **On any conflict between this summary and the rule files, the rule files win**; keep this summary in sync when the rules change.

## Do this first — read the authoritative rules

Before doing anything else, Read these and follow them exactly:

- **The planning workflow** → [ai-rules/negotiation/spec-plan.md](ai-rules/negotiation/spec-plan.md) — the orchestration rule for this workflow.
- **The artifact it writes** → [ai-rules/reference/build-plan-format.md](ai-rules/reference/build-plan-format.md) — plan sections, JSON block envelope, secrets handling, and the query-params-vs-body rules.
- **What it plans** → [ai-rules/negotiation/api.md](ai-rules/negotiation/api.md), [ai-rules/negotiation/query.md](ai-rules/negotiation/query.md), [ai-rules/negotiation/flow.md](ai-rules/negotiation/flow.md), [ai-rules/negotiation/test-group.md](ai-rules/negotiation/test-group.md) — every planned artifact must be structurally valid under these.
- **Wiring** → [ai-rules/reference/context-wiring.md](ai-rules/reference/context-wiring.md) — `context_export` / `context_import` / `header_import` / `{{context.*}}` formats.
- **Env vars and secrets** → [ai-rules/negotiation/env-vars.md](ai-rules/negotiation/env-vars.md) — placement, syntax, `${SECRET:VAR}` handling, which environment a planned variable belongs in.
- **Reads** → [ai-rules/safeguard.md](ai-rules/safeguard.md).

Present every choice per [ai-rules/reference/selection-format.md](ai-rules/reference/selection-format.md).

## The non-negotiable constraints (full detail is in the files above)

1. **Plan everything, build nothing.** No write tool runs under this skill — not `create_or_update_api`, `save_query`, `create_flow`, `create_test_group`, `manage_environment_variables`, `manage_prerequisite_template`, no `validate_*`, no run tool. The only output is the local plan file.
2. **Source first.** No spec in the request → the FIRST response asks for it, zero tool calls. Skip the ask when it was provided.
3. **Parse locally, then ONE read-only resolution pass** — exactly `list_api_definitions`, `get_test_cases` (matched APIs), `list_queries`, `list_flows`, `list_test_groups`, `list_prerequisite_templates` (+ `get_prerequisite_template` on plausible matches), `get_environment`/`list_environments`. Nothing else. A match needs content agreement, not just a name. **`list_*` output is capped at 20k chars and truncates silently**, so a listing proves a name is *taken*, never that it is *free* — confirm every name you intend to `create` with a targeted `get_api_definition`/`get_query`/`get_flow`/`get_test_group`/`get_prerequisite_template` (404 = genuinely free).
4. **One grouped question per section, in order** — APIs → prerequisites → queries → journeys/flows → test group → plan settings. **Everything belonging to one API is asked in one message** (name, env-var placements, test cases, assertions, tags/priority) as a preset with a Custom escape; the same per query, per journey, for the group. Only the target environment, a blocking gap, and the final confirmation stand alone. Never one parameter per message, never two sections in one message, never dump the ledger.
5. **Existing artifacts are reported as existing.** An API that already matches the spec's cURL is named back to the user and reused; missing spec scenarios are offered as **additive** test cases. A same-name-different-content collision always resolves to a new name — never an overwrite.
6. **Query string, body, and path segments stay separate.** The endpoint carries no `?`; query pairs become `params`, the body becomes `payload` with the cURL's real `content_type`, and `<PLACEHOLDER>` path markers resolve to an env var, a flow input, or an earlier step's export — never left literal.
7. **Auth resolves three ways:** an existing prerequisite template (expanded and confirmed, never confirmed by name alone), a new template via `manage_prerequisite_template` planned from the spec's auth cURL, or in-flow login wiring (`context_export` → `header_import`, `prefix: "Bearer "`). No template ever stores a credential the user did not ask to store. No auth source at all → blocking gap; never guess a login endpoint.
8. **A flow packages a journey, not a call.** Composition is settled in its own question *before* any per-flow question: prefer **one flow for the whole spec**, split into subflows + one master flow only for a stated reason (independently runnable/reportable, reused, incompatible inputs/session config, or past ~12–15 steps). A flow whose entire content is one `api_call` wrapping one API is a defect.
9. **Context wiring is derived automatically**, not asked open-ended: every consumed `{{context.<step>.<field>}}` traces to an earlier export, a flow input, or `{{context.env.*}}`; each flow's question shows imports ← producers and exports → consumers for confirmation; renames rewrite downstream prefixes immediately; the value-producing test case is listed **last** (exports are last-wins). Packaging changes the wiring — re-derive the whole graph after the composition decision instead of carrying per-journey wiring over, and state plainly when steps are independent rather than inventing a dependency.
10. **Unautomatable is documented, not invented around.** Client-side-only journeys and spec-declared out-of-scope items are excluded and recorded in Coverage with their reason. Never invent an endpoint the spec does not contain.
11. **Secrets never enter the plan file** — `${SECRET:VAR}` references only, values masked, resolved by the executor at build time (`env-vars.md` §5).
12. **Finish with the alignment check** — re-read the **spec** (not the ledger) and verify every journey, every acceptance criterion, every cURL, and every ordering constraint is accounted for, and that nothing was planned the spec did not ask for. Repair mismatches before presenting.
13. **Stop at the plan.** Present the path + counts + coverage summary and ask whether to change anything or hand it to the execution skill. **Never chain into building.**

If you find yourself about to call a write tool, stop — that is the other skill's job.

## When this skill does NOT apply

- Fully-specified steps document (per-step endpoints/SQL/wiring) → **qa-flow-flow-import**.
- Postman collection / OpenAPI spec → **qa-flow-collection-import**.
- One cURL, or a goal described informally → **qa-flow-flow-negotiation**.
- Executing a plan this skill already wrote → **qa-flow-plan-execute**.
- Information-only ("summarize this spec", "what does it cover?") → read locally and answer; no resolution pass, no plan.

## Session resilience (any assistant)

- Spec planning can outlive the context window. If you can no longer see the rule file's full text or the internal ledger, **re-read `spec-plan.md`** and restate the configured-sections checkpoint before continuing (safeguard Rule 7).
- If the client has a native single-question picker (e.g. AskUserQuestion), prefer it for every section question and the final confirmation — still one question per message.
