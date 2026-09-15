# QA Flow — Spec Plan Rule (Spec Document → Build Plan)

## Scope
These rules apply to **any AI assistant** whenever a user asks to turn a **test spec / feature spec / E2E spec** into QA Flow automation — a document that describes *journeys, acceptance criteria, and the APIs they call* (typically with cURLs), rather than pre-bound per-step technical instructions.

This rule produces **one artifact: a build plan file** (`reference/build-plan-format.md`). It **creates nothing in QA Flow** — no APIs, no queries, no flows, no test groups, no env vars. Building is the job of `operations/plan-execute.md`, invoked separately, from the plan this rule wrote.

**One spec in → two outputs**: the build plan written by this rule, and the artifacts built later by the execution rule from that plan.

### Which rule owns this document?

| The document is… | Use |
|---|---|
| Journeys + acceptance criteria + cURLs per journey; business intent, not per-step bindings | **this rule** |
| Fully-specified numbered steps, each already bound to an endpoint/SQL with `{{context.*}}` wiring and timeouts | `negotiation/flow-import.md` |
| A Postman collection / OpenAPI spec — endpoints with no journey | `negotiation/collection-import.md` |
| A goal, informally described ("test that login works") | `negotiation/flow.md` |

A spec that contains a fully-specified steps section for one journey is still this rule's territory — parse that journey per `reference/steps-doc-format.md` idioms, and keep the rest of the spec under this rule.

This rule **composes with** `negotiation/api.md`, `negotiation/query.md`, `negotiation/flow.md`, and `negotiation/test-group.md` — every artifact it plans must be structurally valid under them. It substitutes their per-entity question cadence with *parse → resolve → grouped per-section negotiation → plan → alignment check*. All MCP reads obey `safeguard.md`.

---

## Core Principle — Plan Everything, Build Nothing

**Never call a write tool under this rule.** Not `create_or_update_api`, not `save_query`, not `create_flow`, not `create_test_group`, not `manage_environment_variables`, not `validate_flow`, and no run tool. The only permitted MCP calls are the read-only resolution pass in Step 3. The only write is the **local build-plan file** — an ordinary file write, not an MCP call.

The spec ledger is the **agent's internal working state, never a deliverable**: do not display the full ledger or a whole-spec plan mid-negotiation. The user sees **one grouped question per message**, and at the end, the finished plan.

Always go through the steps below in order.

---

## Step 1 — Get the Source (Only If Missing)

If the request does not point to a spec (a file path, an attachment, or pasted text), the first response is — with **zero tool calls**:

> *"Point me at the spec to plan — an E2E/feature spec with its journeys and the cURLs they call."*

**Skip this entirely when the source is already provided.** Never re-ask for what is in the request.

---

## Step 2 — Parse the Spec Into the Internal Ledger (Local Read — No MCP)

Read the file locally and extract, in one pass:

1. **Journeys** — each journey's id/title, the acceptance criteria it covers, its positive step list, and its negative/edge scenarios.
2. **APIs** — every cURL in the document. For each, parse **method, host/path, query string, headers, body, and content type as separate fields** per `reference/build-plan-format.md`'s *Query Params vs Body* section. A query string stays out of the endpoint; a path placeholder stays out of both.
3. **Queries** — every SQL / NoSQL statement or DB verification the spec mentions, with its parameters.
4. **Auth needs** — every request carrying an `Authorization`/credential header, and whatever the spec says about how auth is obtained (e.g. *"login via `_shared/`, OTP at runtime"*).
5. **Ordering** — any stated execution order between journeys.
6. **Out of scope / deferred** — everything the spec itself declares blocked, deferred, or unavailable.
7. **Config literals** — base URLs, tokens, tenant ids, codes, and any `<PLACEHOLDER>` markers.

Record it in the **internal ledger**, one row per API, query, journey, and coverage item, each with its open gaps. Derive the candidate names (`snake_case`) for every artifact now, so the resolution pass can check them.

**Classify each journey's automatability while parsing:**

- **Automatable** — the journey's assertions can be made against an HTTP response or a DB row.
- **Partially automatable** — some scenarios are reachable, others are blocked (missing backend data, an unavailable environment). Record *which* scenario and *why*.
- **Not automatable** — the journey is client-side UI only (rendering, taps, navigation, RTL mirroring), or the spec itself declares it out of scope. Record the reason, quoting the spec where the spec says so.

**A not-automatable journey is excluded from the build, never silently dropped** — it lands in the plan's Coverage section with its reason, and is stated in the Step 6 alignment check. Never invent an endpoint to make a UI-only journey look automated, and never plan an API call that the spec does not contain.

Run the resolution pass (Step 3) **before replying** — the first response to an eligible spec is the Step 4 question for the first section, never a bare summary or the raw ledger.

---

## Step 3 — Resolution Pass (the Only Permitted MCP Calls in This Rule)

One read-only batch, exactly these, nothing else:

1. `list_api_definitions` — match every parsed cURL against existing API definitions, and name-check every candidate.
2. `get_test_cases` — on each **matched** API, to see which of the spec's scenarios already exist as test cases.
3. `list_queries` — match each SQL/NoSQL statement, and name-check the candidates.
4. `list_flows` — name-check every journey's candidate flow name.
5. `list_test_groups` — name-check the candidate group name.
6. `list_prerequisite_templates` — find templates that could satisfy the spec's auth needs; `get_prerequisite_template` on any plausible match to read its expanded steps.
7. `get_environment` / `list_environments` — which target environments exist, and which `{{env.*}}` the spec references already exist there (a truncated listing proves presence only — `negotiation/env-vars.md` §6).

**No write tool, no `validate_*`, no run tool, and no other read tool.** Anything beyond this enumerated batch is the forbidden "gathering context first" pattern.

### ⚠️ A `list_*` result may be TRUNCATED — confirm every name it says is free

Tool results are capped (20,000 chars) and the cap is silent apart from a trailing `... [result truncated at N chars]` marker. On a mature project `list_api_definitions` and `list_queries` **routinely truncate** — in this repo's own dashboard, 14 of 50 APIs fall past the cut, including `login_with_otp` and `csrf_demo`. An artifact hidden by truncation looks **free**, which would make the plan create a duplicate of something that already exists, and miss a reuse the user wanted.

So: a listing proves a name is **taken**, never that it is **free**.

**Before writing any `action: "create"` block, confirm the name with a targeted lookup** — `get_api_definition(name)` / `get_query(name)` / `get_flow(name)` / `get_test_group(name)` / `get_prerequisite_template(name)`. These are the reliable oracle: they return the artifact when it exists and raise a 404 error when it does not. This targeted per-candidate check is part of the permitted resolution batch, not an extra context-gathering pass.

If the listing was truncated, say so when reporting what exists, and never claim "no API matches this endpoint" from a truncated list alone — endpoint matching also needs the targeted lookups.

A match requires **content agreement**, not just a name: same method + endpoint for an API, same statement + params for a query. Same name with different content is a **collision**, not a match.

Annotate every ledger row: ✅ existing match / ❌ new / ⚠️ collision or near-match / ⛔ blocked.

---

## Step 4 — Negotiate Section by Section (ONE Grouped Question Per Message)

Walk the ledger in this order: **APIs → prerequisites → queries → journeys/flows → test group → plan-level settings.** Prefer the client's native structured picker (`AskUserQuestion`; see `reference/selection-format.md`) for every question.

### The grouping rule — one message per section, not per parameter

**Every decision belonging to one API section is asked in a single message.** For one API that means, together in one question: its name (with the availability result), which env vars its config literals and placeholders become, its test-case scenarios drawn from the spec's positive and negative paths, its assertions, and its tags/priority — presented as a **preset** the user accepts with one answer, plus a Custom escape. Picking the preset is explicit confirmation of every value in it (`reference/selection-format.md`, *Grouped low-priority parameters*).

The same holds per query, per journey, and for the test group: **one section, one message.**

What must still stand alone as its own question (never folded into a section):

- The **target environment** the plan will be built against.
- Any **blocking gap** (see below).
- The **final plan confirmation** (Step 6).

### Per-section question content

**API section (per endpoint).** State what the resolution pass found, then ask the grouped question:

- **✅ Existing match** — *tell the user it already exists* and show its method + endpoint and its current test cases. Default: reuse as-is. The grouped question covers whether to reuse, and whether the spec's scenarios that are missing from it should be **added as test cases** (additive — never touching its existing cases).
- **❌ New** — the grouped question proposes the derived name (stated as available), the endpoint with the query string split into `params`, the body as `payload`, the resolved content type, each config literal's placement per `negotiation/env-vars.md` §3 (env var recommended for anything environment-specific or secret; a variable an API references is planned in the Default environment), each `<PLACEHOLDER>`'s resolution (env var / flow input / earlier step's export), the test cases derived from the spec's scenarios with their expected statuses, the assertions the spec's expected results imply, and tags/priority.
- **⚠️ Collision** — same name, different content. A collision **always** resolves to a new name; the question offers 2–3 derived alternatives plus free text. Never overwrite.

Show the parsed request back as method, endpoint, params, and body **as separate lines**, so a mis-split is caught here rather than at build time.

**Prerequisite section (per API that needs auth).**

1. If `list_prerequisite_templates` found a template that fits, present it — its name and its **expanded steps and mappings** read via `get_prerequisite_template` — and ask the user to confirm using it. A name alone is never a confirmation (`safeguard.md` Rule 7).
2. If no template fits but the spec provides the auth cURL, plan the **auth API definition** from that cURL and wire it, then ask — in the same grouped message — how the chain should be reused:
   - **A saved prerequisite template** (`manage_prerequisite_template`, action `create`) — recommended when more than one API in the spec needs the same auth, or when the user will run these APIs individually. The plan block carries the template's `steps`, its `injections` (source field → destination header, with prefix), and `carry_cookies`.
   - **Inside the flows only** — the auth API becomes each flow's **first step** with `context_export` + `header_import`, and no template is created. Enough when auth is only ever needed mid-flow.
   - **Both** — a template for single-API runs *and* the in-flow login step.

   Confirm the resolved mapping either way — which response field lands in which header, with what prefix. **Never plan a template that stores a credential value** (a password, an OTP, a token) unless the user explicitly asks for it: the template store is committed with the project. Credentials belong in env vars referenced by the auth API's test case.
3. If neither exists — no template, no auth cURL in the spec — it is a **blocking gap**: ask for the auth source before continuing. Never guess a login endpoint.
4. A runtime-interactive step the spec names (e.g. *"OTP at runtime"*) is surfaced explicitly: ask whether it comes from a flow input, a fixed test credential, or blocks the journey.

**Query section (per statement).** Grouped question per `negotiation/query.md`: name (with availability), the statement, all parameters in one table, return type, and the database it targets. Statements that are identical across journeys share **one** query — say so rather than planning duplicates.

**Flow composition (ONE question, BEFORE any per-flow question).** A flow packages a **journey** — an ordered set of calls that share a context — not a single API call. **One flow per API call is a defect**: it orchestrates nothing, hides the journey the spec actually describes, and pushes the ordering that belongs *inside* a flow out into the test group.

Decide the packaging explicitly, and show the reasoning:

1. **Prefer ONE flow for the whole spec.** If every planned call can run as one ordered sequence — sharing auth/setup, or simply independent with no conflicting inputs or session config — package them into a single flow whose step order **is** the spec's stated ordering. This is the default: it is what makes the spec's journey executable end to end in one run.
2. **Split into subflows + ONE master flow** only when a stated reason applies — name which:
   - a journey must be **independently runnable or reportable** (tagged, released, or re-run on its own);
   - a journey is **reused** by another spec or flow;
   - journeys need **different `inputs` or `session_config`** that cannot coexist in one flow;
   - one flow would grow past a readable step count (~12–15 steps).

   Each subflow packages one journey's steps; the master flow invokes them in the spec's ordering as `flow` steps (`{"type": "flow", "flow_name": …}` — `reference/step-conditional-and-flow.md`), passing values down through `flow_inputs` and pulling results back through `context_export`. Subflows are built **before** the master flow, and `depends_on` must say so.
3. **Never** plan a flow whose entire content is one `api_call` step wrapping one API, unless the user explicitly asks for it. Such an API is run directly, or folded into the flow that needs it.

Present the candidate packaging(s) with the step count each flow would carry and the reason for any split, as its own grouped question, before any per-flow question.

**Journey/flow section (per flow, after composition is settled).** One message per flow the composition produced — a single packaged flow means a single message, not one per journey — carrying: the flow name (with availability), the ordered steps mapped from the spec's positive paths plus its negative/edge scenarios, which API/query/subflow each step uses, the flow inputs, shared-session config, and — explicitly — **the wiring**.

> **Automatic context wiring.** Derive every step's `context_export` and each consumer's `context_import` / `header_import` / `{{context.*}}` template **automatically** from what the steps need, per `reference/context-wiring.md`; do not make the user invent them. Present the derived wiring in the journey's question as a table of *imports ← producer* and *exports → consumers* for confirmation, not as an open question. Rules that always hold:
> - A consumed `{{context.<step>.<field>}}` must trace to an earlier step's `context_export`, a flow input, or `{{context.env.*}}` — a dangling reference is a plan defect, repaired before the plan is written, never carried into it.
> - `context_export` is a **list**; DB results are read as `result.<field>`; `header_import.variable` uses the short form with no `context.` prefix.
> - Auth tokens flow through `header_import` with `prefix: "Bearer "` — never a hardcoded token.
> - Renaming a step rewrites every downstream `{{context.<oldname>.*}}` reference ledger-wide, immediately.
> - When a step selects several `test_cases`, the case producing the exported value is listed **last** (exports are last-wins).
> - **Packaging changes the wiring — re-derive it, never carry it over.** Calls that would have been separate flows share one context once packaged together. After the composition decision, walk the whole step order once and state, per step, what it consumes and from which earlier step. A step that exports a value no later step reads exports nothing; a step that consumes a value no earlier step produces is a defect, not a TODO.
> - **Across a master/subflow boundary**, a value must be exported by the subflow **and** declared in the master's `flow` step `context_export` to reach the parent; `flow_inputs` carries values the other way. Consumed across the boundary with neither is a dangling reference.
> - **Independent steps are wired to nothing, and that is a real answer** — say so explicitly in the flow's wiring table rather than inventing a dependency to justify the packaging.

Scenarios the spec marks unreachable are **not** planned as steps; they go to Coverage as ⚠️ partial with the reason.

**Test group section.** One message: the group name (with availability), the flows in the spec's stated ordering, stop-on-failure, tags, and release.

**Plan-level settings.** The target environment (its own question), the plan file path, and anything the spec left ambiguous that is not owned by a section above.

### Loop discipline

One section per message, in order; each answer updates the ledger; then the next section's question. Never batch two sections into one message, never present the remaining ledger as a plan, never call any tool during the loop. The loop ends when every ledger row is configured and the wiring graph has no dangling edges.

---

## Step 5 — Write the Build Plan (Local File)

Write the plan to `test-plans/<spec-slug>.build-plan.md` exactly per `reference/build-plan-format.md`: every required section in order, every artifact as prose + a complete fenced `json` block, secrets as `${SECRET:VAR}` only, the Coverage table accounting for **every** journey and acceptance criterion in the spec, a Build Order derived from `depends_on`, and Open Questions.

Then run the file's **Validity Checklist** against what you wrote. A failing item is fixed before the plan is presented — never reported as a caveat.

---

## Step 6 — Spec Alignment Check, Then Confirm (ONE Message)

Before presenting the plan, verify it back against the **source spec**, not against the ledger. Re-read the spec and check:

1. **Every journey** in the spec appears in Coverage exactly once, as covered / partial / not automatable.
2. **Every acceptance criterion** id the spec names is accounted for in Coverage.
3. **Every cURL** in the spec became either a planned API or a documented reuse — none dropped, none invented.
4. **Every ordering constraint** the spec states is reflected in the flows' step order and the test group's item order.
5. **Everything the spec declares out of scope** is excluded and recorded with the spec's own reason.
6. **Nothing is planned that the spec does not ask for** — no invented endpoints, scenarios, or assertions.

A mismatch is repaired before presenting, not disclosed as a known gap.

Then present **one** message: the plan's file path, the counts (N env vars, M queries, K APIs, F flows, 1 group), what is reused vs created, the coverage summary (`X covered · Y partial · Z not automatable`, with the not-automatable ones named and reasoned), any Open Questions, and the confirmation:

> *"The build plan is at `test-plans/<slug>.build-plan.md` — review it. Want me to change anything, or should I hand it to the execution skill to build?"*

If the user amends anything, update the plan file and re-present. **This rule stops here.** Building is a separate, explicitly-invoked step under `operations/plan-execute.md` — never chain into it automatically.

---

## What This Rule Is Not

- It is not a builder. It calls no write tool and creates nothing in QA Flow — its only output is the plan file.
- It is not `flow-import.md`. That rule imports one fully-bound steps document into one flow; this one turns a journey-level spec into a whole build plan across APIs, queries, flows, and a group.
- It does not produce a **run plan** (`reference/run-plans.md`) — a build plan is never handed to `user_runner.py`.
- It does not permit dumping the internal ledger — one grouped question per section, then the finished plan.
- It does not automate what the spec says cannot be automated, and it does not invent endpoints to fill a UI-only journey.

> **On any conflict between this file and `api.md` / `query.md` / `flow.md` / `test-group.md` / `safeguard.md`, those rules win for the concern they own.** This file owns only the spec → build-plan orchestration.
