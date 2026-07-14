# QA Flow — Flow Negotiation Rule

## Scope
These rules apply whenever a user requests the creation or update of a **flow** using the QA Flow MCP tools (`create_flow`, `update_flow`).

---

## Core Principle — Negotiate Before You Build

**Never call `create_flow` or `update_flow` as the first response to a user request.**

**Never respond to a flow request by presenting a concluded summary or "proposed plan" covering all steps at once. That is not negotiation — it skips the user entirely.**

Negotiation means: ask one question, wait for the answer, ask the next question. The user drives the decisions. You surface the options.

Always go through the negotiation steps below. Each flow-building concern has its own reference file — read the relevant one before asking questions or building.

| Concern | Reference File |
|---|---|
| Flow JSON top-level structure, Flow Inputs (runtime parameters) | `ai-rules/reference/flow-structure.md` |
| Shared Session Config (`session_config` headers) and Environment Variables (`env.*` in payloads/headers) | `ai-rules/reference/context-wiring.md` |
| API selection and test case discovery | `ai-rules/reference/api-and-testcase-selection.md` |
| `api_call` step — fields, payload merge, multi test-case | `ai-rules/reference/step-api-call.md` |
| `wait_until` step — API/DB polling, conditions | `ai-rules/reference/step-wait-until.md` |
| `query` step — saved DB queries, result access | `ai-rules/reference/step-query.md` |
| `conditional`, `flow`, and `wait` steps | `ai-rules/reference/step-conditional-and-flow.md` |
| All step types quick reference | `ai-rules/reference/build-workflow.md` |
| Context export, import, header_import, templates | `ai-rules/reference/context-wiring.md` |
| File downloads from step responses | `ai-rules/reference/file-downloads.md` |
| Build sequence and common mistakes | `ai-rules/reference/build-workflow.md` |

---

## ⛔ CRITICAL — NO Step Type Is Exempt From Negotiation

**This rule applies to ALL step types without exception:** `api_call`, `query`, `wait_until`, `conditional`, `flow`, `wait`.

The following behaviors are **strictly forbidden**, regardless of step type or how much detail the user already provided:

- Calling `list_queries`, `list_api_definitions`, `get_available_apis`, `get_flow`, `get_test_cases`, or **any** MCP tool **before starting the negotiation sequence** — with a single exception: the one `list_flows` name-availability check that immediately precedes the 2a question (see 2a).
- Calling `create_flow` or `update_flow` without completing Steps 1–3 below.
- Treating a stated step type, query name, API name, or flow name as sufficient to skip negotiation.
- "Gathering context first" — this is not preparation, it is the forbidden pattern itself.

**The source comes first, then the name check.** If the request doesn't describe what the flow should do (its goal / the steps to chain), the first response asks exactly that — with zero tool calls. As soon as the purpose is known, the **only permitted pre-negotiation action** runs: the 2a name-availability check — derive the intended flow name from the purpose, call `list_flows` once to check whether it already exists, then immediately ask the 2a name question with that result in hand. Zero other tool calls before 2a is answered.

---

## Step 1 — Gather Context from the MCP Server (After 2a Is Answered)

After the user confirms the flow name and description, **then** query the server:

1. `list_flows` — already called for the 2a name check; reuse its result: does a similar flow exist? Is this an update or a new flow?
2. `list_api_definitions` — which APIs are available as steps? Do all required APIs exist?
3. If a similar flow exists, call `get_flow` on it to understand existing patterns (step types, context wiring, session config, naming). Use this as **reference only** — never assume the new flow should copy it.
4. Call `get_test_cases` on **every API** that will be used as a step, so you can present the available test cases to the user.
5. If steps reference DB queries, call `list_queries` to verify the query methods exist.

Use this context to inform your questions, not to skip them.

---

## Step 2 — Negotiate Each Step With the User (Always, Even When APIs Exist)

**The existence of a similar flow or a known API does NOT allow you to skip negotiation.**
The user may want different test cases, different exports, different headers, a different step order, or none of the defaults from a similar flow.

Work through each concern below by asking the user directly. Do **not** assume — even for things that seem obvious.

### 2a — Purpose, Then Flow Name and Description
- **The purpose comes first.** If the user's request doesn't describe what the flow should do (its goal or the steps to chain), ask exactly that as the first question — zero tool calls: *"What should this flow do — which calls or checks should it chain, end to end?"* Skip this entirely when the request already describes it — never re-ask for what was provided.
- **Then run the name-availability check** — the only pre-negotiation tool call: detect the intended flow name (the user's stated name, or a concise `snake_case` proposal derived from the purpose, e.g. `login_with_otp`, `checkout_flow`, `verify_user_identity`) and call `list_flows` to see whether it already exists.
- **Name is free** → ask: **"I suggest naming this flow `<proposed_name>` — '<proposed_description>'. Does that work, or would you like a different name or description?"**
- **Name already exists** → a collision **always** resolves to creating a new flow under a **new name** — never overwrite or update the existing flow as a fallback (updating is a separate, explicit user request). Tell the user it exists and offer **2–3 alternatives** (versioned suffix like `login_with_otp_v2`, or a more specific purpose word) plus a free-text option, e.g. `(1) login_with_otp_v2  (2) login_with_url_otp_check  (3) type another name`.
- Keep the description to one sentence explaining what the flow does end-to-end.
- Wait for the user to confirm or provide their own name/description before moving on; if their free-text name also collides, re-check before continuing.

### 2b — Steps — Per-Step-Type Negotiation Questions

For **every step** in the requested flow, work through its type-specific questions **one at a time, in order**. Complete all questions for step N before moving to step N+1.

---

#### Step Type: `api_call`

Ask these in order — two single questions, then two **grouped** questions (per the *Grouped low-priority parameters* rule in `ai-rules/reference/selection-format.md`):

1. **Which API?** Call `list_api_definitions` first. Present as numbered menu: `(1) csrf  (2) login_with_otp  (3) get_client_api_groups`
2. **Which test case?** Call `get_test_cases` on the chosen API. Present all available test cases as a numbered menu. Never default to `default` without asking.
3. **Overrides & delays — ONE grouped message.** Does this step need any of: **(a)** payload overrides beyond what the test case provides (field names and values — static, `{{context.*}}`, `{{auto.*}}`); **(b)** URL query-param overrides (`params`); **(c)** `waitBefore` / `waitAfter` delays (default 0)? Offer "none" as the easy answer. **When a payload field or query param matches an earlier step's export (by name or purpose), pre-suggest that `{{context.<step>.<field>}}` value** per *Smart Context Wiring* below. **For every static literal value, do not silently hardcode it — apply *Static Values → Flow Inputs or Env Vars* below.**
4. **Context wiring — ONE grouped message.** **Lead with the resolved suggestion** per *Smart Context Wiring* below — matched export→consumer pairs and import candidates as a picker (accept all / subset / none / custom). Cover: **(a)** What to export: which response fields should this step save to context for downstream steps (numbered menu including a "none" option). **(b)** What to import as headers (`header_import`): header name, context variable (e.g. `csrf_step.csrfToken`), and optional prefix (e.g. `"Bearer "`) — or none.

---

#### Step Type: `query`

Ask these in order — the parameters are collected as ONE grouped message:

1. **Which saved query?** Call `list_queries` first. Present all available query methods as a numbered menu — the menu itself confirms existence. **If the needed query does not exist**, it must be created with `save_query` (under `ai-rules/negotiation/query.md`) and smoke-tested before the flow is built.
2. **Query parameters — ONE grouped message.** Present **all** required parameters in a single table-style question — for each, the value comes from `(a) flow input {{context.flow_input.X}}  (b) env var {{context.env.X}}  (c) previous step {{context.step_name.field}}  (d) static value`:

   ```
   `get_user_status` takes 2 parameters — where should each come from?
   - user_id: (a) flow input  (b) env var  (c) previous step  (d) static
   - tenant:  (a) flow input  (b) env var  (c) previous step  (d) static
   ```

   **Lead with inputs/env vars, not the static option** — if the user gives a static literal, apply *Static Values → Flow Inputs or Env Vars* below before keeping it hardcoded. When an earlier step already exports a matching value, **pre-suggest it** for option (c) — name the exact context variable (e.g. `create_user.userId` for `user_id`) per *Smart Context Wiring* below.
3. **What to export?** Which fields from the query result should be saved to context? DB results are accessed as `result.<field>` — for arrays use `result[0].<field>`. Apply *Smart Context Wiring* below: present inferred exportable fields matched to their downstream consumers as a picker, plus a "none" option.

---

#### Step Type: `wait_until`

Ask these in order — cadence + condition and the wiring are each ONE grouped message:

1. **What to poll — API or DB query?**
   - If API: same questions as `api_call` items 1 and 2 (which API, which test case).
   - If DB query: same questions as `query` items 1 and 2 (which query, what params — grouped).
2. **Polling configuration — ONE grouped message: timeout + interval + condition.** Offer preset cadences plus custom, and ask for the condition in the same message:

   ```
   Polling cadence: (1) Quick — 30s timeout, 2s interval  (2) Default — 60s timeout, 5s interval
   (3) Patient — 300s timeout, 15s interval  (4) Custom — tell me timeout / interval
   And what condition ends the wait? (a) visual — field path, operator (==, !=, contains, is_not_empty), expected value
   (b) python_code — describe the logic and I'll write the snippet
   ```

   **Remind the user:** DB query results must use `result.<field>` as the condition field path, never bare field names.
3. **Context wiring — ONE grouped message.** **Lead with the resolved suggestion** per *Smart Context Wiring* below. **(a)** What to export from the final successful poll response. **(b)** *(API-polling variant only — do NOT skip it just because this is a `wait_until` step)* what to import as headers (`header_import`): e.g. Step 1's `csrfToken` as an `X-CSRF-Token` header, or an auth/bearer token — header name, context variable, optional prefix.

---

#### Step Type: `conditional`

Ask these in order — two grouped messages:

1. **The condition — ONE grouped message (mode + expression).**
   - If visual: which context field to check (e.g. `login_step.accessToken`), which operator (`==`, `!=`, `>`, `<`, `contains`, `is_empty`, `is_not_empty`), and what value to compare against. If multiple conditions, the logic operator: `(1) AND  (2) OR`. **Pre-suggest the likely field** per *Smart Context Wiring* below — offer the exports already agreed on earlier steps as pickable candidates (and if the needed field isn't exported yet, propose retro-adding that export).
   - If python_code: ask the user to describe the logic; you will write the snippet using `get_var()`, `continue_flow()`, `raise_error()`, `stop_flow()`, `run_subflow()`, `jump_to_step()`.
2. **Outcomes — ONE grouped message: on-true AND on-false together.** Present the options once and ask for both branches: `(1) continue  (2) warn  (3) stop  (4) raise_error  (5) run_flow (pick sub-flow)  (6) jump (pick step name)` — e.g. *"On true → ? On false → ?"*. **If `run_flow` is chosen for a branch**, follow up (for that branch only): which sub-flow, and what input values to pass (`subflow_inputs`).

---

#### Step Type: `flow` (sub-flow step)

Ask these in order — the wiring is ONE grouped message:

1. **Which sub-flow to invoke?** Call `list_flows` first. Present as numbered menu.
2. **Wiring — ONE grouped message: inputs + exports.** Call `get_flow` on the chosen sub-flow to check its declared `inputs`, then present **all** of them in one table-style question — for each, where does the value come from: flow input, env var, previous step context, or a static value? **Apply *Smart Context Wiring* below: when an earlier step's export matches a sub-flow input (by name or purpose), pre-suggest that exact context variable as the recommended pick for it.** **Offer inputs/env vars first; if the user gives a static literal, apply *Static Values → Flow Inputs or Env Vars* below before hardcoding it.** In the same message, ask which context keys should be pulled from the sub-flow back into the parent flow (or none).

---

#### Step Type: `wait`

Ask only one question:

1. **How long to wait?** How many seconds? Should the delay happen before (`waitBefore`) or after (`waitAfter`) — or both? Note: prefer adding `waitBefore`/`waitAfter` directly on the adjacent step instead, unless this step's sole purpose is the delay.

---

> **Selection format rule:** small fixed choice sets → an inline numbered menu on one line — `(1) a  (2) b  (3) c`. Long lists of existing items → a numbered table/list, one option per line with a short identifying detail; show at most **20** (tell the user more exist and to just name theirs); prefix rows with `- [ ]` when several can be picked. Full rule, examples, and native structured-UI guidance: read `ai-rules/reference/selection-format.md` before presenting your first list.

### 🧠 Smart Context Wiring — Resolve, Suggest, Let the User Pick

Context variables can feed **every step type**: `api_call` headers (`header_import`), payload fields and URL/query params (`{{context.*}}` templates), DB `query` inputs, sub-flow (`flow`) inputs, and `conditional` condition fields. Whenever a step's question set reaches a context-wiring topic (what to export, what to import, where a parameter's value comes from), **do not ask it open-ended — resolve the likely wiring yourself first**, then present the result as a pickable suggestion:

1. **What this step can provide** — infer exportable fields from the chosen API's test cases (`get_test_cases` — assertions and `context_export` in existing cases reveal the response shape), the saved query's result columns, or the sub-flow's exported context keys. Auth/session material (tokens, csrf, ids, GUIDs, status fields) are prime candidates.
2. **What other steps will need** — scan the flow goal and every step already negotiated (and those still coming) for consumers: API headers, payload fields, query params, DB query inputs, sub-flow inputs, conditional comparison fields.
3. **Match producers to consumers** and present the pairs as the suggested wiring inside the step's existing grouped context-wiring question — as a picker (use the native structured picker per `ai-rules/reference/selection-format.md` when available):

   ```
   Suggested context wiring for step 2 (`login`):
   - [ ] export `accessToken` → step 3 `Authorization` header (prefix "Bearer ")
   - [ ] export `userId`      → step 4 query input `user_id`
   (1) accept all suggested  (2) keep only some — tell me which  (3) none  (4) different wiring — tell me
   ```

Rules:

- **Suggestions never replace the question.** The user always confirms; accepting the suggestion (option 1) counts as explicit confirmation of every suggested pair. This stays ONE grouped message per step — the suggestion is the content of the wiring question, not an extra message.
- **Retro-fill missing exports.** While negotiating step N, if it consumes a value no earlier step exports, propose adding the export to the producing step (naming the step and field) — or a flow input / env var per *Static Values → Flow Inputs or Env Vars* — and get the user's pick. Never leave a dangling `{{context.*}}` reference.
- **Mirror agreed wiring on the consumer side.** When the consuming step's question comes up later, present the already-agreed import as the recommended default option instead of re-asking from scratch.
- **Never invent fields.** Suggest only what is evidenced by test cases, query columns, sub-flow exports, or the user's stated goal. If nothing sensible can be inferred, fall back to the plain open question.

### ⚠️ Static Values → Flow Inputs or Env Vars

**Whenever a negotiated value is a static literal — a hardcoded payload field, query param, query input, sub-flow input, `wait_until`/`conditional` comparison value, header value, etc. — do not silently bake it into the flow.** Stop and ask the user where the value should live:

- **Varies between runs or is test data** (user IDs, GUIDs, feature IDs, phone numbers, amounts, emails) → declare it as a **flow input** and reference it with `{{context.flow_input.X}}`. See 2d.
- **Environment-specific or shared config / secrets** (base URLs, client IDs, tenant IDs, status IDs, API keys, tokens) → store it as an **env var** and reference it with `{{context.env.X}}` (or `env.X` in `header_import.variable`). See 2e.
- **Genuinely constant for every run and every environment** → only then keep it hardcoded, and only after the user explicitly confirms that's what they want.

Collect placements efficiently: a single literal is one question; **several pending literals are presented together in ONE table-style message**, each row picking its home — never one message per literal:

```
You gave 3 static values — where should each live?  (1) flow input  (2) env var  (3) keep hardcoded
- tenant = montymobile:      (1) / (2) / (3)   [recommend: env var]
- user_id = 8842:            (1) / (2) / (3)   [recommend: flow input]
- channel = "web":           (1) / (2) / (3)
```

Default recommendation: **flow input** for run-varying values, **env var** for environment/config/secret values. **Never assume hardcoding** — a static literal in a payload, param, or input is a prompt to ask this question, not a value to copy verbatim. This applies across every step type.

### 2c — Shared Session Config
> See `ai-rules/reference/context-wiring.md` for full schema and usage.

**Shared session is enabled by default.** Always set `use_shared_session: true` unless the user explicitly opts out.

- Ask: **"Shared session is on by default — this means all steps will share a common `session_config` (e.g. base headers like `tenant`, `device-id`, `RecaptchaToken`). Should I keep shared session enabled?"**
- If yes (or no answer to the contrary) → set `use_shared_session: true` and ask which headers belong in `session_config`.
- If no → set `use_shared_session: false` and skip `session_config`.
- Tokens extracted mid-flow (e.g. `Authorization: Bearer`) go in `header_import` on the step, not in `session_config`.

### 2d — Flow Inputs
> See `ai-rules/reference/flow-structure.md` (the `inputs` field) for schema and usage.

- Do any values vary between test runs (user IDs, feature IDs, GUIDs)?
- This is also where every static literal flagged by the *Static Values → Flow Inputs or Env Vars* rule lands when the user chooses "flow input."
- If yes → declare them as `inputs` with `name`, `description`, `type`, `required`, `default_value`, and reference them with `{{context.flow_input.X}}` wherever the literal appeared.
- If all values come from env vars (or the user explicitly chose to keep them hardcoded) → no `inputs` needed.

### 2e — Environment Variables
> See `ai-rules/reference/context-wiring.md` (env templates) for full syntax and variable inventory.

- Are any values (base URLs, client IDs, status IDs, tenant IDs, tokens) environment-specific, shared config, or secrets?
- This is also where every static literal flagged by the *Static Values → Flow Inputs or Env Vars* rule lands when the user chooses "env var."
- If yes → use `{{context.env.VAR_NAME}}` in payloads/params, or `env.VAR_NAME` in `header_import.variable`. If the env var does not exist yet, tell the user it must be added to the environment config (via `manage_environment_variables` / the environment) and name the variable you propose.
- Do not hardcode values that belong in environment config or flow inputs — promote them per the rule above.

---

## Asking Order — Sequential, Strictly — ONE TOPIC PER MESSAGE

**This is the most critical rule in this document. Read it carefully.**

Ask **one topic per message**. High-stakes decisions (which API, which test case, the final plan) get their own question; **related low-priority parameters are grouped into a single message** exactly as each step type's question set above specifies (overrides+delays together, exports+imports together, all query params together, polling cadence+condition together — see the *Grouped low-priority parameters* rule in `ai-rules/reference/selection-format.md`). Never combine **unrelated** concerns, options across different steps, or summaries into the same reply. The user must answer before you ask the next topic.

The order per step is: work through the **full question set for that step's type as defined in 2b above** — e.g. the four messages for `api_call` (API, test case, overrides+delays, context wiring), the three for `query` — in the listed order, waiting for each answer. Do **not** stop after the API/test-case questions; the overrides, static-value, and wiring questions are part of the set.

Complete every question for step N before moving to step N+1.
Only after **all steps are fully negotiated**, ask the **flow-level configuration (2c + 2d + 2e) as ONE grouped message**: shared session (on by default — which `session_config` headers?), flow inputs, and env vars. By this point most of it is confirming what already accumulated from the static-value promotions — present it as a short checklist, not three separate interrogations.

### ❌ Forbidden Patterns

- **Never** present a full summary of all steps as a "proposed plan" before each step is individually negotiated.
- **Never** bundle "here are my assumptions + here is my plan" in a single message.
- **Never** mix unrelated topics in the same message (e.g. "Which API should this use, and what should the whole flow export?"). Grouping the **related low-priority parameters of one concern** into a single message (overrides+delays, exports+imports, all params of one query) is *required* by the question sets above — what is forbidden is cross-topic bundling and plan-disguised-as-question.
- **Never** conclude what the user wants and skip asking — even if it seems obvious from context.
- **Never** show a full multi-step plan and call it "negotiation". That is not negotiation.
- **Never** call any MCP tool (`list_queries`, `list_api_definitions`, `get_available_apis`, `get_flow`, `get_test_cases`, or anything else) before the 2a question — **except** the single `list_flows` name-availability check that immediately precedes it. The first *response* must always be the 2a negotiation question, informed by that check.
- **Never** treat any step type as an exception to negotiation. `query`, `api_call`, `wait_until`, `conditional`, `flow`, `wait` — all require the full per-step-type negotiation sequence above before building.
- **Never** skip step-type-specific questions because the step type or target name was stated in the request. The full question set for that step type must be asked.

### ✅ Correct Pattern Example

```
Agent:  Step 1 — which API should we use? Here are the options: ...
User:   csrf
Agent:  Which test case for `csrf`? Available: valid_csrf_token, invalid_csrf_token2
User:   valid_csrf_token
Agent:  Any overrides for step 1 — payload fields, query params, or waitBefore/waitAfter delays? (or "none")
User:   none
Agent:  Context wiring for step 1 — what should it export (e.g. csrfToken, Set-Cookie, or none),
        and what should it import as headers (or none)?
User:   export csrfToken, import nothing
Agent:  Moving to step 2 — which API should we use? Here are the options: ...
```

---

## Step 3 — Present a Full Proposed Plan and Ask for Confirmation

After all steps are negotiated, present a complete summary:

- **Flow name** and description
- **Inputs** declared (name, type, default) — or "none"
- **Session config** headers — or "none"
- **Steps in order**:
  - Step name → type → API/query/flow used → test case
  - What it exports to context (field name and context key)
  - What it imports from context (header_import or template vars)
- **Environment variables** referenced (note any that must still be created in the environment config)
- **Static values promoted** — for each literal the user supplied, state where it landed (flow input, env var, or intentionally hardcoded)
- **Assumptions** you are making (be explicit — list every default you applied)

Then ask: **"Does this look right, or would you like to change anything before I build it?"**

Do not proceed until the user explicitly confirms or provides corrections.

---

## Step 4 — Build

Only after explicit user confirmation:

1. Create any missing API definitions first (following `ai-rules/negotiation/api.md`)
2. Create any missing saved queries first (following `ai-rules/negotiation/query.md`)
3. Call `create_flow` or `update_flow` with the agreed parameters
4. Call `validate_flow` before executing
5. Follow all rules in `ai-rules/safeguard.md`

---

## Step 5 — After Creating the Flow, Ask to Run It

After `create_flow` or `update_flow` completes successfully, **always** ask the user if they want to run the flow before doing anything else.

Use this exact pattern — one message, one question:

```
The flow "[flow_name]" has been created successfully. Would you like to run it now?
```

- If the user says **yes** → run the flow test suite using `run_test_suite` (or the appropriate run tool).
- If the user says **no** → stop and wait for further instructions.
- Do **not** run the flow automatically without asking.
- Do **not** combine this question with any other message or suggestion.

---

## Step Naming Conventions

- Use descriptive names that reflect the action: `get_csrf_token`, `login_with_url_otp`, `form_step`
- Or use positional names when order is more important than the action: `step_1`, `step_2`, `step_3`
- Be consistent within a single flow — don't mix both styles

---

## What This Rule Is Not

- It is not a rigid questionnaire. Questions are adapted to the specific request — but they are never skipped.
- It does not block quick edits where all details are already explicitly stated by the user.
- It does not apply when the user is asking for information only (e.g. "show me the flow", "list all flows").
- It does not govern creating a flow from a **fully-specified steps document** (per-step endpoints/SQL, `{{env.*}}`/`{{context.*}}` wiring, timeouts) — `negotiation/flow-import.md` replaces the per-step question cadence there with an internal ledger driving a per-step confirmation loop plus one summary + build confirmation; this file still owns the flow's structure and step semantics.
- A similar existing flow is **reference material only** — it is never a substitute for asking the user what they want.
