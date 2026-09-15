# QA Flow — Build Plan Format (Spec → Plan → Execution)

A **build plan** is the artifact `negotiation/spec-plan.md` produces and `operations/plan-execute.md` consumes. It is the frozen, reviewable record of a negotiated spec import: everything that must be created, everything that already exists and will be reused, how the journeys wire together, and what the spec asked for that could not be automated.

> ⚠️ **A build plan is not a run plan.** `reference/run-plans.md` and `docs/plan-format-reference.md` describe **run plans** — YAML/Python files that orchestrate *already-built* artifacts in the Docker runner. A build plan describes **what to create** through MCP write tools. They never mix: a build plan is never passed to `user_runner.py`, and a run plan is never passed to the executor rule.

---

## Location and Naming

```
test-plans/<spec-slug>.build-plan.md
```

- `<spec-slug>` derives from the source spec's filename or title (`e2e-spec.md` → `e2e-spec`; `MS-4633 Bundle UX` → `ms-4633-bundle-ux`).
- Create the `test-plans/` directory if it does not exist. Writing this file is a **local file write, not an MCP call** — it is permitted (and required) before any build confirmation.
- Never overwrite an existing build plan silently. Same spec re-planned → write `<spec-slug>.build-plan.v2.md` and say so, or amend in place only on the user's explicit instruction.

---

## Shape — Prose Sections, Each Carrying a Fenced `json` Block

Every artifact section is **human-readable prose followed by one fenced ` ```json ` block** holding the exact MCP call. The prose is what the user reviews; the JSON block is what the executor replays. They must agree — the executor trusts the JSON and never re-derives payloads from the prose.

Each JSON block is a single object with these envelope keys:

| Key | Required | Notes |
|---|---|---|
| `id` | yes | Stable identifier unique within the plan (`api.get_bundle_faq`, `flow.j1_bundle_faq`, `query.faq_rows`, `env.AUTH_TOKEN`, `group.ms_4633_e2e`). Used by `depends_on` and by the executor's checkpoint ledger. |
| `action` | yes | `create` (artifact is missing) or `reuse` (artifact exists and is used as-is). A `reuse` block carries no `tool`/`args` — only `id`, `action`, `existing`, and a `verified` note. |
| `tool` | on `create` | The exact MCP tool name: `manage_environment_variables`, `save_query`, `create_or_update_api`, `manage_prerequisite_template`, `create_flow`, `create_test_group`. |
| `args` | on `create` | The complete argument object for that tool — exactly what the executor passes, with no edits. |
| `depends_on` | no | List of `id`s that must be built first. The executor topologically orders the build from these. |
| `covers` | no | Spec identifiers this artifact serves (`["FAQ-01", "FAQ-02"]`, `["J1"]`). Feeds the Coverage section and the alignment check. |

A `create` block's `args` must be **complete and directly executable** — no placeholders like `"<fill at build>"`, no partial payloads, no `TODO`. Anything still unknown at plan time means the planning loop is not finished; ask the question rather than deferring it into the plan.

---

## Required Sections, In Order

### 1. Header

Spec source path, spec title, plan date, target environment name and `BASE_URL` env var, and a one-line statement of what the plan builds (counts: N env vars, M queries, K APIs, F flows, 1 test group).

### 2. Environment Variables

One block per env var the plan needs. **Values are never written into the plan** — see *Secrets* below. Syntax, placement, naming, the `sensitive` flag, and which environment each variable must exist in follow `negotiation/env-vars.md`; a variable that needs a different value per environment gets one block per environment.

```json
{"id":"env.AUTH_TOKEN","action":"create","tool":"manage_environment_variables",
 "args":{"env_id":"00000000-0000-0000-0000-000000000000","action":"add","name":"AUTH_TOKEN",
  "value":"${SECRET:AUTH_TOKEN}","category":"Custom","sensitive":true},
 "covers":["J2","J3"]}
```

### 3. Saved Queries

One block per SQL/NoSQL query, conforming to `negotiation/query.md`. Include the statement, its parameters, and the return type. Steps sharing identical SQL share **one** query block, referenced by both flow steps.

### 4. API Definitions

One block per endpoint, conforming to `negotiation/api.md`. Query string and request body are **separate, explicitly typed** fields — never merged (see *Query Params vs Body* below). Each block lists its test cases with `expected_status`, assertions, tags, and priority.

A `reuse` block for an existing API records what was verified against the spec's cURL:

```json
{"id":"api.get_bundle_faq","action":"reuse",
 "existing":{"name":"get_bundle_faq","method":"GET","endpoint":"{{env.BASE_URL}}/v2/faq"},
 "verified":"method+endpoint match the spec cURL; test case `faq_bundle_empty` exists",
 "covers":["FAQ-04"]}
```

### 5. Prerequisites / Auth Chains

For each API that needs auth, one entry stating how it is satisfied, with the resolved mapping — which response field lands in which header, with what prefix:

- **`reuse`** — an existing prerequisite template, named, with its expanded steps as read by `get_prerequisite_template`.
- **`create`** — a new template built by `manage_prerequisite_template`, when the chain is reused across APIs or the user runs APIs individually.
- **in-flow only** — no template; the auth API is each flow's first step, wired with `context_export` + `header_import`. Recorded here as prose so the reader can see the decision, with the wiring itself living in the flow block.

```json
{"id":"prereq.login_chain","action":"create","tool":"manage_prerequisite_template",
 "args":{"action":"create","name":"esim_login","description":"Login for staging eSIM APIs",
  "steps":[{"type":"api","name":"login_v1","test_case":"valid_login"}],
  "injections":[{"source_step":1,"path":"response.data.accessToken",
                 "destination":"header","target":"Authorization","prefix":"Bearer "}],
  "carry_cookies":true},
 "depends_on":["api.login_v1"],"covers":["J2","J3"]}
```

**A template block never stores a credential value.** No password, OTP, or token in `steps[].inputs` — the template store is committed with the project. Credentials stay in env vars read by the auth API's own test case. If the user explicitly asks for a stored value, it is still written as `${SECRET:VAR}` and resolved at build time.

### 6. Flows

One block **per flow**, conforming to `negotiation/flow.md` and `reference/flow-structure.md`. The prose lists the steps in order with their spec step numbers; the JSON block is the complete `create_flow` argument object including `steps`, `context_export` / `context_import` / `header_import` wiring, `inputs`, and `session_config`.

**A flow packages a journey, not a call.** How many flow blocks a plan carries is the composition decision made during negotiation (`negotiation/spec-plan.md`, *Flow composition*) — normally **one flow for the whole spec**, and a master flow plus subflows only when a stated reason demands it. A block whose `steps` array is a single `api_call` wrapping one API is a plan defect: it orchestrates nothing. When the plan does split, each subflow gets its own block, the master flow invokes them as `{"type": "flow", "flow_name": …}` steps, and every subflow's `id` appears in the master's `depends_on` so the executor builds them first.

Every flow block is preceded by a **wiring table** naming each step's imports and exports with their producers and consumers:

| Step | Imports | Exports | Consumed by |
|---|---|---|---|
| `login` | — | `accessToken` | `fetch_faq`, `fetch_bundle` |
| `fetch_faq` | `login.accessToken` (header `Authorization`, prefix `Bearer `) | `faqCount` | `assert_hidden` |

### 7. Test Group

One block bundling the flows in the spec's stated ordering, conforming to `negotiation/test-group.md`.

### 8. Coverage

A table mapping **every** journey and acceptance criterion in the spec to its automation status. This section is the input to the spec-alignment check and to the executor's post-build report.

| Spec ID | Covered by | Status | Note |
|---|---|---|---|
| FAQ-04 | `flow.j1_bundle_faq` → `assert_faq_empty` | ✅ covered | |
| FAQ-01 | — | ⚠️ partial | populated-accordion path is backend-data-blocked (`category=bundle` unseeded) |
| GE-01…GE-05 | — | ⛔ not automatable | client-side UI only, no endpoint |
| X-01 | — | ⛔ not automatable | spec marks analytics out of scope, no product-defined event |

Statuses: `✅ covered` · `⚠️ partial` (with what is missing) · `⛔ not automatable` (with the reason, quoting the spec where the spec itself declares it out of scope).

### 9. Build Order

The dependency-ordered `id` list the executor follows, derived from `depends_on`:

```
env.AUTH_TOKEN → env.BUNDLE_CODE → query.faq_rows → api.login_v1 → api.get_bundle_faq
→ prereq.login_chain → flow.j1_bundle_faq → flow.j2_supported_countries → group.ms_4633_e2e
```

### 10. Open Questions

Anything the user explicitly deferred, and anything the executor must ask before building. An empty section reads `None — the plan is complete.` A non-empty section means the executor **stops and asks** before its build confirmation.

---

## Secrets

**Secret values never appear in a build plan.** The plan file is a repo artifact meant to be reviewed and committed.

- A secret is referenced as `${SECRET:VAR_NAME}` inside `args`, and listed in the Environment Variables section as `VAR_NAME: ••• (provided at build)`.
- The executor resolves `${SECRET:VAR_NAME}` from the process environment at build time; if it is unset, the executor **asks the user for the value in that build's conversation** and never writes it back into the plan file.
- What counts as a secret is defined in `negotiation/env-vars.md` §5 (tokens, passwords, API keys, DB credentials, any `Authorization` value, credentials inside a URI). Base URLs, tenant names, and non-sensitive IDs are ordinary env vars and may carry their literal value in the plan. A secret's block sets `"sensitive": true`.
- A cURL's inline `Authorization: Bearer eyJ…` is **never** copied into the plan — it becomes `{{env.AUTH_TOKEN}}` (or a `header_import` from a login step) at plan time.

---

## Query Params vs Body — Get This Right at Plan Time

The single most common import defect is folding a URL query string into the request body, or vice versa. The plan must keep them separate and explicitly typed, because `create_or_update_api` treats them as different fields:

| Source in the cURL | Plan field | Example |
|---|---|---|
| `?category=bundle&page=2` in the URL | `params` (per test case) / `query_params` (per flow step) | `"params": {"category": "bundle", "page": "2"}` |
| `-d '{"phone":"..."}'` with `application/json` | `payload` | `"payload": {"phone": "{{context.flow_input.phone}}"}` |
| `-F key=value` / file upload | `payload` + `content_type: multipart/form-data` | |
| `-d 'a=1&b=2'` with a form content type | `payload` + `content_type: application/x-www-form-urlencoded` | |
| `/v2/bundle/<BUNDLE_CODE>` — a path segment | **neither** — it is part of `endpoint`, templated: `{{env.BUNDLE_CODE}}` or `{{context.flow_input.bundle_code}}` | |

Rules the plan must satisfy:

1. **The `endpoint` in the plan carries no query string.** Strip `?…` off the URL and move every pair into `params`. An endpoint that still contains `?` is a plan defect.
2. **Path placeholders are templated, never left as `<PLACEHOLDER>`.** The spec's `<BUNDLE_CODE>` / `<TOKEN>` style markers each resolve during negotiation to an env var, a flow input, or a value exported by an earlier step.
3. **A GET has no body.** If a spec's GET cURL shows `-d`, flag it in Open Questions rather than guessing.
4. **`content_type` is set from the cURL**, not assumed — `application/json` only when the request actually sends JSON.
5. **Query values are strings** in `params`; numbers and booleans are written as their string form.

---

## Worked Fragment

```markdown
## API — `get_bundle_faq`  [NEW]  · covers FAQ-01…FAQ-04

`GET {{env.BASE_URL}}/v2/faq?category=bundle` — fetches the FAQ items the bundle
details page renders. Query param `category` is split out of the URL; no body.
Two test cases: the staging default (empty list, section hidden) and the
populated path (recorded as partial — backend data blocked).

```json
{"id":"api.get_bundle_faq","action":"create","tool":"create_or_update_api",
 "args":{"name":"get_bundle_faq","method":"GET",
  "endpoint":"{{env.BASE_URL}}/v2/faq","content_type":"application/json",
  "test_cases":[
   {"test_case":"faq_bundle_empty","description":"category=bundle returns no items; UI hides the section",
    "params":{"category":"bundle"},"expected_status":200,
    "assertions":[{"field":"status_code","operator":"==","value":200},
                  {"field":"response.totalCount","operator":"==","value":0}],
    "tags":["smoke","positive"],"priority":"high"}]},
 "depends_on":["env.BASE_URL"],"covers":["FAQ-01","FAQ-02","FAQ-03","FAQ-04"]}
```
```

---

## Validity Checklist

A build plan is well-formed only when all of these hold. The planner verifies them before presenting the plan; the executor re-verifies them before building and refuses a plan that fails any:

- [ ] Every section above is present, in order.
- [ ] Every JSON block parses, and carries `id` + `action` (+ `tool` + `args` on `create`).
- [ ] Every `id` is unique; every `depends_on` entry names an `id` that exists in the plan.
- [ ] Build Order is a valid topological order of the `depends_on` graph — no cycles, nothing missing.
- [ ] No `args` value contains a literal secret; secrets appear only as `${SECRET:…}`.
- [ ] No `endpoint` contains a `?` query string; no unresolved `<PLACEHOLDER>` markers anywhere.
- [ ] Every `{{context.<step>.<field>}}` in a flow block traces to an earlier step's `context_export`, a flow input, or `{{context.env.*}}`.
- [ ] No flow block's `steps` is a single `api_call` wrapping one API (a flow that orchestrates nothing), and every `{"type": "flow"}` step names a flow the plan builds earlier or that already exists.
- [ ] Every journey and acceptance criterion in the source spec appears exactly once in Coverage.
- [ ] Open Questions is either `None` or explicitly flagged as blocking.
