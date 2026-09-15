# QA Flow Report Dashboard Negotiation Rule

## Scope
These rules apply to **any AI assistant** whenever a user requests a **write or run action in the reporting / analytics dashboard domain** (the dashboard's 📊 **Reports** tab) using the QA Flow MCP tools. The report dashboard surfaces seven things; these are the **negotiated** actions:

| # | Dashboard feature | MCP tool(s) | Effect |
|---|---|---|---|
| 1 | Tag runs with a release | `manage_releases action='tag'` | Re-labels existing job runs (persistent) |
| 2 | Untag runs | `manage_releases action='untag'` | Removes the release stamp |
| 3 | 🚀 **Release Execution** | `test_group_releases action='by_release'` → `execute_test_group` per group | **Fires real test executions** against the active environment |
| 4 | 🔄 **Run Fresh & Compare** | `test_group_releases action='execute_and_compare'` | **Re-runs both releases' groups**, then compares (long-running) |
| 5 | 📤 **Export Reports** | `export_test_group_results` / `get_load_test_results export=true` | Writes a report/result file to disk |
| 6 | Clear the analytics cache | `clear_analytics_cache` | Discards cached analytics |

The reporting dashboard is a **read model** over the SQLite **jobs database** — every metric (overview, trends, per-flow/suite/api stats, error analysis, release comparison) is derived from existing job records. Most write tools here only **re-label** runs (tagging) or **invalidate the cache**; but **Release Execution** and **Run Fresh & Compare** actually **execute tests** — treat those with the same care as running a test group or load test.

> **Two distinct "release" systems live here — never conflate them:**
> - **Job-level release tag** (`manage_releases` tag/untag) — writes `release_version` into an individual run's `payload`. Powers the **analytics** release comparison (`manage_releases action='compare'`).
> - **Test-group release** (`test_group_releases`) — a `release` field set on a *test group* when it's created/updated. Powers the dashboard's 🚀 **Release Execution** and 🔄 **Release Comparison** dropdowns. The release names in those dropdowns are test-group releases, **not** job tags.

The read-only companions are **not** governed by this rule and are used freely, before or without negotiation:

- `get_analytics` — every report (`overview | metrics | trends | flows | suites | errors | api-stats | api-trends | health`).
- `manage_releases` `action='list'`/`'compare'`/`'untagged_jobs'` and `test_group_releases` `action='list'`/`'by_release'`/`'compare'` (the **cached** comparison, no re-run).
- `get_test_group_results` / `get_load_test_results` (no `export`) — viewing execution and ⚡ **Load Test Results** history.
- `get_job_history` / `get_run_status` — to discover the exact `job_id`s a tag/untag will target.

> **Information-only requests never trigger this rule.** "Show me the overview", "compare v1.1 and v1.2 (cached)", "which runs are untagged", "list releases", "show the load test results" are read actions — answer them directly. Negotiation begins only when the user wants to **tag**, **untag**, **execute a release**, **run-fresh-and-compare**, **export a file**, or **clear the cache**.

---

## Core Principle — Negotiate Before You Mutate or Run

**Never call `manage_releases` (`tag`/`untag`), `test_group_releases action='execute_and_compare'`, `execute_test_group` (for a release run), `export_test_group_results`, `get_load_test_results export=true`, or `clear_analytics_cache` as the first response to a user request.**

**No tool calls before negotiation begins.** The FIRST response to such a request must be the Step 2 *action* question — **not** `manage_releases`/`test_group_releases` (`list`/`by_release`), **not** `get_job_history`, **not** `get_analytics`, **not** `health_check`, nor any other read-only tool. "Gathering context first" is the forbidden pattern itself. Context-gathering happens in Step 1, **after** the action is confirmed.

Two of these actions are **execution**: 🚀 Release Execution runs every test group in a release, and 🔄 Run Fresh & Compare re-runs **both** releases' groups — real traffic against the active environment, results written into the jobs DB. Release **tagging** is persistent and silently destructive — it overwrites a run's stored `release_version`, which is exactly what comparison reads. Treat all of these with the care of any outward-facing action. Negotiate every target, version, and environment; never assume them.

Negotiation means: ask one question, wait for the answer, ask the next. The user drives the decisions; you surface the options.

---

## Step 1 — Gather Context from the MCP Server (After the Action Is Confirmed)

Only **after** the user has answered the Step 2 action question, query the server to build the menu and catch problems early:

1. **Tag / untag** → list candidate runs so the user picks real `job_id`s:
   - `manage_releases action='untagged_jobs'` (default `limit` 50) for runs **not yet tagged**.
   - `get_job_history` for the full run list (to re-tag, untag, or tag already-tagged runs).
   - `manage_releases action='list'` to show which job-level release versions already exist.
2. **🚀 Release Execution** → `test_group_releases action='list'` to show release names, then `action='by_release'` on the chosen release to list exactly which **test groups** will run. Surface the group count and, if needed, `list_environments` to confirm which environment the runs hit.
3. **🔄 Run Fresh & Compare** → `test_group_releases action='list'` for the two release names; optionally `action='compare'` first to show the **current cached** picture before deciding to spend a full re-run.
4. **📤 Export** → `get_test_group_results` (an execution to export) or `get_load_test_results` (a load-test execution to export) to confirm the exact `execution_id` exists.
5. **Clear cache** → no discovery required; only confirm intent.

Use this context to inform your questions, not to skip them. **Never invent a `job_id`, a release name, or an `execution_id` the user hasn't confirmed.** If the user names something that doesn't appear in the listings, surface that instead of guessing.

---

## Step 2 — Ask One Question at a Time (Sequential, Strictly)

**Critical rules:**
- **One topic per message.** High-stakes decisions (the action, the runs/groups affected, the environment) get their own question; **related low-priority parameters are grouped into a single message** per the *Grouped low-priority parameters* rule in `ai-rules/reference/selection-format.md`. Never bundle unrelated concerns or the whole plan into one reply.
- **Never skip a question** because the user gave a name or because a default exists.
- **Never present a summary or proposed plan** until every question for the chosen path is answered.
- **After each answer, ask only the next question — nothing else.**

Ask in this exact order, one message per question. The first question routes to one of six paths:

1. **Action — what do you want to do in the report dashboard?** Offer the write/run actions as a numbered list (laid out per the Selection format rule) — `(1) tag runs with a release version  (2) remove a release tag  (3) 🚀 execute a release (runs all its test groups)  (4) 🔄 run fresh & compare two releases  (5) 📤 export a report or load-test result to a file  (6) clear the analytics cache` — and ask which. This is the first response — **no tool calls** before it. (If the user only wants to *view* a report, *compare releases from cache*, *list releases*, or *view load-test results*, that is read-only — do it directly, this rule does not apply.)

**Tag path (action 1):**

2. **Which runs?** Run Step 1 discovery, then present the candidate runs as a numbered list (laid out per the Selection format rule) — e.g. `Untagged runs: (1) login_flow [job_a1b2 · 2026-06-28 · completed]  (2) checkout_suite [job_c3d4 · failed]`. Ask which run(s) to tag. **Flag any selected run that is already tagged** — tagging overwrites the existing release with no warning.
3. **Release version + hidden-runs scope — ONE grouped message.** Ask for the version string — it must match `^[a-zA-Z0-9._-]+$` (letters, digits, `.`, `-`, `_` — e.g. `v1.2.0`, `release-2026_q2`); spaces or other characters are rejected with a 400. In the same message, ask whether to include hidden runs in the candidate list (`include_hidden`, default **false** — hidden jobs are excluded from the default `untagged_jobs` listing and from every report).

**Untag path (action 2):**

2. **Which runs?** Present currently-tagged candidate runs (`get_job_history`, noting each run's current `release_version`) as a numbered list (laid out per the Selection format rule). Ask which run(s) to untag. Note untag only affects runs that currently carry a tag — others are silently skipped.

**🚀 Release Execution path (action 3):**

2. **Which release?** Present release names from `test_group_releases action='list'` as a numbered list (laid out per the Selection format rule). Ask which release to execute.
3. **Confirm the groups.** Show the test groups returned by `action='by_release'` (the exact set that will run) and the count. Ask the user to confirm this is the intended set — **these are real test executions, run one group at a time.**
4. **Which environment?** Confirm the environment / `BASE_URL` the runs will hit (`environment_id`; the dashboard uses the active environment). The host is where all the traffic lands — confirm it deliberately.

**🔄 Run Fresh & Compare path (action 4):**

2. **Which two releases? — ONE grouped message.** Present the release names from `test_group_releases action='list'` (laid out per the Selection format rule) and ask for **release A and release B together** (e.g. "reply with two numbers: A, B").
3. **Which environment?** Confirm the environment both re-runs execute against. Remind the user this **re-runs every group in BOTH releases** before comparing — it is long-running and doubles the execution cost vs a cached compare.

**📤 Export path (action 5):**

2. **Export what?** A numbered list (laid out per the Selection format rule) — `(1) a test-group execution report  (2) all executions (Excel)  (3) a load-test result`. Ask which.
3. **Which one / which format?** For a single execution: confirm the `execution_id` (from Step 1) and the format — `json | html | csv | pdf` (test-group) or `json | csv | html` (load test). Excel exports **ALL** executions, not one — call that out. Ask where to save (`save_to`) or accept the default path.

**Clear-cache path (action 6):**

2. **Confirm intent.** Clearing only discards the in-memory analytics cache (5-minute TTL); it changes **no** run data and forces the next report to recompute. (Note: tag/untag already clear the cache automatically — a standalone clear is only needed when run data changed by other means.)

> **Selection format rule:** small fixed choice sets → an inline numbered menu on one line — `(1) a  (2) b  (3) c`. Long lists of existing items → a numbered table/list, one option per line with a short identifying detail; show at most **20** (tell the user more exist and to just name theirs); prefix rows with `- [ ]` when several can be picked. Full rule, examples, and native structured-UI guidance: read `ai-rules/reference/selection-format.md` before presenting your first list.

---

## Step 3 — Present a Proposed Plan and Ask for Confirmation

Only after **all questions for the chosen path are answered**, present the full summary:

- **Action**: tag / untag / release execution / run-fresh-and-compare / export / clear cache
- **Targets**: runs (`name [job_id · status]`) · or the release + its group list · or release A vs B · or the `execution_id` + format
- **Release version** (tag): the exact validated string; for re-tags, the **current → new** version
- **Environment** (execution / run-fresh): the env name and resolved `BASE_URL` the runs hit
- **Scope flag** (tag): `include_hidden`; **Save path** (export): `save_to` or default
- **Side effects you flagged** — e.g. *"executes 4 test groups for real against staging"*, *"run-fresh re-runs both releases (8 groups total) before comparing"*, *"Excel export covers ALL executions, not just this one"*, *"job_c3d4 is already tagged `v1.1`; tagging overwrites it to `v1.2`"*
- **Assumptions** you applied (list every default)

Then ask the path-appropriate confirmation:
- Execution / run-fresh: **"This will run real tests against [BASE_URL]. Does this look right, or change anything before I start?"**
- Tag/untag: **"This re-labels stored runs and affects release reporting. Proceed?"**
- Export: **"This writes a [format] file to [path]. Proceed?"**
- Clear cache: **"This discards the cached reports and forces a fresh recompute. Proceed?"**

Do not proceed until the user explicitly confirms. If the user requests a change, apply only that change and re-present.

---

## Step 4 — Apply

Only after explicit user confirmation:

1. Re-confirm the **targets exist** (`get_job_history` / `test_group_releases action='by_release'` / `get_test_group_results` / `get_load_test_results`). If a named target is missing, stop and surface it — never fabricate an id or name.
2. Call the tool(s) with the agreed parameters:
   - **Tag:** `manage_releases action='tag'` with non-empty `job_ids` **and** `release_version`.
   - **Untag:** `manage_releases action='untag'` with non-empty `job_ids`.
   - **🚀 Release Execution:** `test_group_releases action='by_release'` to get the group list, then `execute_test_group` for **each** group (background by default → capture each `job_id`), passing the agreed `environment_id`. Run them in order.
   - **🔄 Run Fresh & Compare:** `test_group_releases action='execute_and_compare'` with `release_a`, `release_b`, and `environment_id` (long-running).
   - **📤 Export:** `export_test_group_results` (group_name, execution_id, format[, save_to]) or `get_load_test_results execution_id=… export=true`.
   - **Clear cache:** `clear_analytics_cache` (no parameters).
3. Follow all rules in `ai-rules/safeguard.md`.

---

## Step 5 — After Applying, Report the Real Result and Offer the Next Step

**Tag / untag** silently skip runs they can't match, so the returned count can be lower than what you sent. **After the call returns, echo the actual result** — e.g. *"Tagged 2 of 3 requested runs with `v1.2` (1 id wasn't found)."* — and if the count is lower, say so and offer to re-check the ids.

For **execution** and **run-fresh-and-compare** (long-running background work), report the captured `job_id`s, then ask — as its own message, one question — whether to monitor:

```
Started execution of [n] test group(s) for release "[release]".
Would you like me to monitor the runs and report when they finish?
```
- If **yes** → poll `get_run_status` per job; when done, summarize and (for run-fresh) show the comparison. **Then always offer to export the results** — as its own message, one question (laid out per the Selection format rule):
  - **🚀 Release Execution** → the per-group execution reports: *"Would you like to export any group's results? Supported formats: JSON, CSV, HTML, PDF."* → `export_test_group_results(group_name, execution_id, format[, save_to])`.
  - **🔄 Run Fresh & Compare** → the release comparison: *"Would you like to export the comparison? Supported format: CSV."* → the release-comparison CSV export.
- For **export** (action 5) → report the saved file path. Offer nothing further unless asked.

In all cases, do **not** auto-run another action or chain a second execution without asking.

---

## Established Patterns — Apply These When Mutating or Running

### Data model — the report dashboard is a read model over the jobs DB
- Every analytics metric is computed **only** from the SQLite `jobs` table. With no matching runs, reports return **zeros**, not an error.
- `job_type` ∈ `flow | test_group | test_suites`. Dates are `YYYY-MM-DD`; `days` is clamped to `1..365`.
- 🚀 Release Execution and 🔄 Run Fresh & Compare **write new job records** as they run — so they also change what subsequent reports show.

### The two release systems (do not conflate)
- **Job-level tag** (`manage_releases`): `release_version` inside a run's `payload`. Tagging overwrites silently; unknown `job_id`s are skipped (reconcile the returned count); `release_version` must match `^[a-zA-Z0-9._-]+$`; `job_ids` must be non-empty.
- **Test-group release** (`test_group_releases`): the `release` field on a *test group*, set when the group is created/updated under the test-group rule. `action='by_release'` lists the groups carrying that release — that set is what 🚀 Release Execution runs.

### Release Execution & Run Fresh & Compare are real executions
- **Release Execution** = one `execute_test_group` per group in the release, run sequentially against the active environment (background jobs). Same gravity as running a test group; confirm the environment.
- **Run Fresh & Compare** (`execute_and_compare`) re-runs **both** releases' groups, then compares — long-running and roughly double the cost of executing one release. Offer the **cached** compare (`action='compare'`, the dashboard's "Use cached (24h)" toggle) first when the user just wants the numbers.

### Export
- `export_test_group_results` formats: `json | html | csv | pdf` for one execution; **`excel` exports ALL executions** (different endpoint) — never assume excel means "this one".
- Load-test export is `get_load_test_results export=true` (formats `json | csv | html`, scope `full | api_endpoints`). Viewing ⚡ Load Test Results without `export` is read-only and exempt.

### Cache behaviour
- Analytics responses are cached in-memory with a **5-minute TTL** (`CACHE_EXPIRATION = 300`). `tag`/`untag` **auto-clear** it, so release reports update immediately after a tag change — a standalone `clear_analytics_cache` is only needed when run data changed by means other than tagging.
- Test-group release **comparison** has its own **24-hour cache** (the "Use cached (24h)" toggle) — "Run Fresh & Compare" bypasses it by re-running.

### Hidden runs
- `include_hidden` defaults **false** everywhere: hidden runs are excluded from reports, release listings, and the `untagged_jobs` candidate list. A hidden run can still be tagged/untagged by `job_id`, but won't appear in default menus.

---

## What This Rule Is Not

- It is not a rigid questionnaire. Questions are adapted to the path (tag / untag / execute / run-fresh / export / clear) — but they are never skipped, and execution targets, release versions, and environments are never assumed silently.
- It does not block a quick action where the user has **already stated every detail explicitly** ("tag job_a1b2 with v1.2", "export execution exec_9 as pdf") — but you still confirm the target exists, flag side effects, and echo the plan before applying.
- It does **not** apply to information-only requests — viewing any `get_analytics` report, listing releases, comparing releases from cache, viewing test-group or load-test results, or listing untagged jobs are read actions handled directly without negotiation.
- A prior tag, release, or comparison is **reference material only** — never a substitute for asking the user what they want this time.
