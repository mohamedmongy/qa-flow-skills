# QA Flow Load Test Negotiation Rule

## Scope
These rules apply to **any AI assistant** whenever a user requests that a **load test** (a.k.a. stress test / performance test) be started or managed using the QA Flow MCP tools (`start_load_test`, `manage_load_test_queue`).

A load test drives [Locust](https://locust.io) against an **already-existing** target — a `test_group`, a single `flow`, or a single `api`/test suite — by spawning concurrent virtual users for a fixed duration. A load test **never creates** the thing it tests; the flow, API, test suite, or test group must already exist.

The read-only companions — `get_load_test_status` (poll the queue / a job) and `get_load_test_results` (list / fetch / export results) — are not write tools and are used *after* negotiation, never before it.

---

## Core Principle — Negotiate Before You Run

**Never call `start_load_test` or `manage_load_test_queue` as the first response to a user request.**

**No tool calls before negotiation begins.** The FIRST response to a load-test request must be the Step 2 *target* question — **not** `list_test_groups`, `list_flows`, `get_available_apis`, `get_load_test_status`, `health_check`, or any other read-only tool. "Gathering context first" is the forbidden pattern itself. Context-gathering happens in Step 1, **after** the target question is answered.

A load test is **destructive in effect**: it fires real traffic — 100 concurrent users by default — at a real host (`BASE_URL`). Treat starting one with the same care as any outward-facing action. Negotiate every parameter; never assume the defaults.

Negotiation means: ask one question, wait for the answer, ask the next. The user drives the decisions; you surface the options.

---

## Step 1 — Gather Context from the MCP Server (After the Target Type Is Confirmed)

Only **after** the user has answered the Step 2 target question, query the server to build the menu and catch problems early:

1. If the target is a **test group** → `list_test_groups`, then `get_test_group` on the candidate. **Inspect its item types** — the load-test runner only executes `flow` and `test_suite` items; **`query` items are silently skipped** (see *Edge Cases*). If the group is query-only, surface this before going further.
2. If the target is a **single flow** → `list_flows` (or `get_available_apis`) to confirm the exact flow name exists.
3. If the target is a **single api / test suite** → `get_available_apis` / `list_test_suites` to confirm the suite exists.
4. If an environment will be chosen → `list_environments` to present the options and confirm `BASE_URL` resolves.
5. Check `get_load_test_status` (the queue) **only if** the user asks about contention or wants to manage existing jobs — not as an opening move.

Use this context to inform your questions, not to skip them. **Never invent a target name** — if the flow/api/suite/group does not exist, it must be created first under its own rule (`ai-rules/negotiation/flow.md` for flows, `ai-rules/negotiation/api.md` for APIs/suites, `ai-rules/negotiation/test-group.md` for groups).

---

## Step 2 — Ask One Question at a Time (Sequential, Strictly)

**Critical rules:**
- **One topic per message.** High-stakes decisions (target, environment, plan confirmation) get their own question; **related low-priority parameters are grouped into a single message** per the *Grouped low-priority parameters* rule in `ai-rules/reference/selection-format.md`. Never bundle unrelated concerns or the whole plan into one reply.
- **Never skip a question** because the user gave a number or because a default exists.
- **Never present a summary or proposed plan** until every question below has been answered.
- **After each answer, ask only the next question — nothing else.**

Ask in this exact order, one message per question:

1. **Target — what should be load tested?** Offer the three target kinds as a numbered list (laid out per the Selection format rule) — `(1) an existing test group  (2) a single flow  (3) a single API / test suite` — and ask which. This is the first response — **no tool calls** before it. Once the kind is known, do Step 1, then present the concrete candidates as a numbered list (laid out per the Selection format rule) (e.g. `Test groups: (1) smoke_suite  (2) checkout_group`) and ask which one. **If the named target does not exist**, stop and create it first under its own negotiation rule — never invent a name.
2. **Load profile — users + spawn rate + duration, as ONE grouped question.** Do **not** ask these three one-by-one. Present named presets, each showing all three values, plus a custom option — remind the user this is real traffic against `BASE_URL`:

   ```
   How much load? This is real traffic against the target host.
   (1) Light — 10 users, 2/s ramp, 1m
   (2) Moderate — 50 users, 5/s ramp, 3m
   (3) Default — 100 users, 10/s ramp, 5m
   (4) Heavy — 500 users, 20/s ramp, 10m
   (5) Custom — tell me users / spawn rate / duration
   ```

   Picking a preset is the explicit confirmation of every value in it — the **100-user default is never applied silently**; choosing `(3)` is what consents to it. On `Custom`, the user gives the values in one reply (`users`, `spawn_rate`, `duration` — accepts `30s`, `5m`, `1h`); ask only for missing pieces.
3. **Environment, and queue or run immediately?** Which environment / `BASE_URL` to run against (`environment_id`) — the default active environment, or a specific one from `list_environments`. The host is where all the traffic lands, so confirm it deliberately. In the same message, ask whether to add to the queue (`queue=true`, the default, FIFO single-worker) or start right now. **Note the constraint:** *immediate, non-queued* start (`queue=false`) works **only** for a test-group target — single-flow and single-api targets are always queued. If the user wants a single target to start instantly, that is `manage_load_test_queue` `promote` *after* enqueuing, not `queue=false`.

> **Selection format rule:** small fixed choice sets → an inline numbered menu on one line — `(1) a  (2) b  (3) c`. Long lists of existing items → a numbered table/list, one option per line with a short identifying detail; show at most **20** (tell the user more exist and to just name theirs); prefix rows with `- [ ]` when several can be picked. Full rule, examples, and native structured-UI guidance: read `ai-rules/reference/selection-format.md` before presenting your first list.

---

## Step 3 — Present a Proposed Plan and Ask for Confirmation

Only after **all questions above are answered**, present the full summary:

- **Target**: `test_group → <name>` *or* `flow → <name>` *or* `api → <name>`
- **Load profile**: `users`, `spawn_rate` (users/sec), `duration`
- **Environment**: the env name and the resolved `BASE_URL` it will hit — or "default active environment"
- **Execution mode**: queued (and current queue depth, if known) or immediate
- **Side effects you flagged** — e.g. *"a single-target stress test will create a persistent synthetic group `_stress_<type>_<name>.json`"*, or *"this group contains query items that will be skipped"*
- **Assumptions** you applied (list every default — especially the user count)

Then ask: **"This will fire real load at [BASE_URL]. Does this look right, or would you like to change anything before I start it?"**

Do not proceed until the user explicitly confirms. If the user requests a change, apply only that change and re-present for confirmation.

---

## Step 4 — Start

Only after explicit user confirmation:

1. Re-confirm the **target exists** (`list_test_groups` / `list_flows` / `get_available_apis`). If it is missing, stop and create it under its own negotiation rule first — never fabricate it.
2. Call `start_load_test` with the agreed parameters:
   - **Test-group target:** pass `test_group="<name>"`.
   - **Single-target:** pass `target_type="flow"|"api"` **and** `target_name="<name>"` (omit `test_group`). The server builds and persists a synthetic `_stress_<type>_<name>.json` group on the fly. `target_type` **must** be exactly `"flow"` or `"api"`; `"api"` is executed as a `test_suite` item, so `code_generator/TestCases/test_<name>.py` must exist.
   - Pass `users`, `spawn_rate`, `duration`, and `environment_id` exactly as negotiated.
   - `queue` defaults to `true` (returns a `job_id`). Use `queue=false` **only** for an immediate test-group run that the user explicitly asked to start now.
3. Capture the returned `job_id` (queued mode). Follow all rules in `ai-rules/safeguard.md`.

---

## Step 5 — After Starting, Offer to Monitor (Don't Auto-Poll Into Results)

A load test is a **long-running background job**. After `start_load_test` returns successfully, report the `job_id` and queue position, then ask — as its own message, one question:

```
The load test for "[target]" is queued as [job_id] (position [n]).
Would you like me to monitor it and report when it finishes?
```

- If **yes** → poll `get_load_test_status` (job-scoped). When status is `completed`/`failed`, fetch the matching execution with `get_load_test_results` and summarize. Do **not** fetch results before the job finishes — they won't exist yet. **After summarizing, always offer to export the results** — as its own message, one question: *"Would you like to export these load-test results? Supported formats: JSON, CSV, HTML."* (laid out per the Selection format rule). If the user picks a format → `get_load_test_results(execution_id, export=true, format=…[, save_to])` (scope `full` or `api_endpoints`). If the user declines, stop.
- If **no** → stop and wait. Tell the user they can later check `get_load_test_status`, pull results with `get_load_test_results`, and export them (JSON, CSV, HTML) with `get_load_test_results export=true`.
- Do **not** auto-run, auto-promote, or chain a second test without asking.

---

## Established Patterns — Apply These When Running

### The two target modes
- **Test-group mode** — `test_group="<name>"`. Runs every executable item in the group sequentially per virtual user. Supports `queue=false` (immediate).
- **Single-target mode** — `target_type` + `target_name`. The server auto-generates a persistent synthetic group named `_stress_<target_type>_<sanitized_name>.json` in `code_generator/DataSources/test_groups/`. This file **lingers** after the run — mention it as a side effect; it is not auto-cleaned. Single-target mode is **always queued** (no `queue=false`).

### What actually executes
- The Locust runner executes only `flow` and `test_suite` items. **`query` items are silently ignored** — they never run and never appear in load-test stats. If a chosen group's value is in its queries, a load test is the wrong tool; say so.
- A `flow` target imports `code_generator/APIs/user/<name>.py`; a `test_suite` / `api` target runs `code_generator/TestCases/test_<name>.py` via pytest. Both must already exist on disk (created through their own MCP tools).

### Load parameters and their meaning
- `users` (default **100**) — peak concurrent virtual users. The single most impactful knob; never assume it.
- `spawn_rate` (default **10**) — users started per second until `users` is reached (ramp-up).
- `duration` (default **"5m"**) — total run time; `int` seconds are coerced to `"<n>s"`. Accepts `s`/`m`/`h` suffixes.
- Each virtual user waits `1–3s` between iterations and loops through the group's items in order.
- The runner flags an item as failed when its success rate drops **below 95%**.

### Environment / host resolution
- `environment_id` selects the environment whose `BASE_URL` (and headers/vars) the traffic uses. It is validated when enqueuing — an unknown id returns `404`. Omit it to use the default active environment. If no `BASE_URL` resolves, the run cannot start.

### Queue semantics (`manage_load_test_queue`)
- The queue is **FIFO with a single worker** — queued jobs run one at a time, oldest first.
- `action="promote"` starts a still-queued job **immediately in parallel**, bypassing order — use this to "run a queued single-target now."
- `action="cancel"` cancels a job **only while it is still queued**; a running job cannot be cancelled this way.
- On dashboard restart, any `running` job is marked `failed` (interrupted) — re-enqueue if needed.

### Results
- Results land in `code_generator/TestSuites/load_test_results/locust_<group>_<timestamp>.json` and are retrieved via `get_load_test_results` (list all, fetch one by `execution_id`, or `export=true` to write a file).
- Export formats are `json`, `csv`, `html`; scope is `full` or `api_endpoints`.

---

## What This Rule Is Not

- It is not a rigid questionnaire. Questions are adapted to the request — but they are never skipped, and the user-count default is never applied silently.
- It does not block a quick re-run where the user has *already stated every parameter explicitly* ("re-run smoke_suite, 50 users, 2m, default env, queued") — but you still confirm the target exists and echo the plan before starting.
- It does not apply to information-only requests (e.g. "list load test results", "what's in the queue", "show me execution X") — those use the read tools directly.
- An existing or prior load test is **reference material only** — never a substitute for asking the user what they want this time.
