# QA Flow — Plan Execute Rule (Build Plan → Built Artifacts)

## Scope
These rules apply to **any AI assistant** whenever a user asks to **execute / build / apply a build plan** — the artifact produced by `negotiation/spec-plan.md` and defined by `reference/build-plan-format.md`. Triggers: *"execute the plan"*, *"build the plan"*, *"apply `test-plans/<slug>.build-plan.md`"*, *"go ahead and create everything in the plan"*.

This rule is the **second half of the spec pipeline**: `spec-plan.md` negotiates a spec into a plan and builds nothing; this rule turns that plan into real QA Flow artifacts and negotiates nothing. Together: **one spec in → a plan, then the artifacts**.

> ⚠️ **A build plan is not a run plan.** This rule consumes a *build plan* (what to create, via MCP write tools). Running already-built artifacts — plan files, `user_runner.py`, compose, CI — is `operations/docker-run.md` and `reference/run-plans.md`. Never hand a build plan to the runner, and never hand a run plan to this rule.

Every write it performs obeys `safeguard.md`, and every artifact it creates must be structurally valid under the rule that owns it: `negotiation/api.md`, `negotiation/query.md`, `negotiation/flow.md`, `negotiation/test-group.md`.

---

## Core Principle — The Plan Is the Contract

**The approved plan is the negotiation.** This rule does not re-ask what the plan already settled — that would be re-negotiating a decision the user already made. It also does not improve, extend, or reinterpret the plan: it builds **exactly** what the plan's JSON blocks say, in the plan's Build Order, behind **one** explicit confirmation.

Two symmetrical failures to avoid:

- **Re-negotiating** — walking the plan section by section asking the questions `spec-plan.md` already asked. The user approved a plan so they would not have to answer twice.
- **Improvising** — building something the plan does not contain, "fixing" a payload on the fly, inventing a test case, or silently skipping a block. Anything the plan gets wrong is raised with the user, not patched mid-build.

If the plan is wrong or incomplete, **stop and say so** — the repair belongs in the plan (and usually back in `spec-plan.md`), not in an improvised build.

---

## Step 1 — Load and Verify the Plan (Local Read — No MCP)

Read the plan file locally. If the request names no plan, ask for it — with **zero tool calls** — listing any `test-plans/*.build-plan.md` you can see:

> *"Which build plan should I execute? I can see `test-plans/e2e-spec.build-plan.md`."*

Then run the **Validity Checklist** from `reference/build-plan-format.md` against it. Verify in particular:

1. Every required section is present; every JSON block parses.
2. Every block has `id` + `action`, and `create` blocks have `tool` + `args`.
3. Every `depends_on` names an `id` in the plan; the Build Order is a valid topological order with no cycles.
4. No `args` value holds a literal secret (secrets appear only as `${SECRET:VAR}`); no `args` holds a placeholder like `<fill at build>`.
5. No `endpoint` contains a `?` query string; no unresolved `<PLACEHOLDER>` markers.
6. Every `{{context.<step>.<field>}}` in a flow block traces to an earlier step's `context_export`, a flow input, or `{{context.env.*}}`.
7. **Open Questions is empty.** A non-empty Open Questions section **blocks the build**: present those questions, get the answers, write them into the plan file, then continue.

A plan failing any check is **refused, not repaired silently**: report exactly which check failed and where, and offer to fix the plan file (or to hand it back to `spec-plan.md`). Never build from a plan that fails verification.

---

## Step 2 — Pre-Flight Resolution (Read-Only MCP)

The plan may have been written days ago; the server may have moved. Before confirming, run a read-only pass to detect drift — exactly these:

1. `list_api_definitions`, `list_queries`, `list_flows`, `list_test_groups`, `list_prerequisite_templates` — does each `create` block's name still not exist, and does each `reuse` block's target still exist?
2. `get_environment` / `list_environments` — does the target environment exist, and which of the plan's env vars (and `${SECRET:…}` values) are already set? A truncated variable listing proves presence only — confirm absence per `negotiation/env-vars.md` §6.
3. `health_check` — is the dashboard reachable (this rule's writes all go through it)?

**`list_*` results are capped at 20,000 chars and truncate silently** (apart from a trailing `... [result truncated]` marker), so a listing proves a name is **taken**, never that it is **free**. Confirm every `create` block's name and every `reuse` block's target with a **targeted lookup** — `get_api_definition` / `get_query` / `get_flow` / `get_test_group` / `get_prerequisite_template` — which returns the artifact or raises a 404. Building a `create` whose name only *looked* free is exactly the collision this pre-flight exists to prevent.

Classify each block:

- **✅ ready** — `create` whose name is free, or `reuse` whose target still exists unchanged.
- **⚠️ drifted** — a `create` whose name now exists (someone built it meanwhile), or a `reuse` whose target changed or disappeared.
- **⛔ blocked** — a missing environment, an unreachable dashboard, an unresolvable dependency.

**Drift is surfaced in the confirmation, never resolved unilaterally.** A `create` whose name now exists is *not* overwritten (`safeguard.md`, and every negotiation rule's collision stance): the confirmation offers **skip it if the existing one matches the plan** / **create under a new name** / **stop**. Never pass an existing name to a create tool and call it an update.

Resolve `${SECRET:VAR}` values now, per `negotiation/env-vars.md` §5: read each from the environment, and for any that is unset, **ask the user for the value in this conversation** (one grouped question naming all missing secrets; never echo a value). Secrets are written `sensitive: true`. Never write a resolved secret back into the plan file.

---

## Step 3 — Build Manifest + ONE Confirmation

Present **one** message containing the compact build manifest — not the plan re-printed:

1. **Plan** — file path, source spec, target environment and its `BASE_URL`.
2. **Will create** — counts and names by type: N env vars, M queries, K APIs, P prerequisite templates, F flows, 1 test group.
3. **Will reuse** — existing artifacts the plan depends on (names only).
4. **Drift** — every ⚠️ block, with the resolution being proposed for it.
5. **Secrets** — which env vars will be set, with values masked (`AUTH_TOKEN: •••`), and where each came from (environment vs. just supplied).
6. **Build order** — the ordered `id` list about to run.
7. **Coverage** — the plan's one-line summary (`X covered · Y partial · Z not automatable`).
8. **The confirmation question**, naming the blast radius:

> *"Build it? One yes authorizes, in order: 3 env vars → 1 query (smoke-tested) → 4 APIs → 1 prerequisite template → 4 flows (each validated) → 1 test group. Nothing is run afterwards."*

Per `safeguard.md`, the confirmation is its own message and must name the exact targets and the environment. **One yes authorizes the whole chain** — no per-artifact re-confirmation during the build. If the user amends anything, update the plan file and re-present the manifest.

**Extra confirmations still required inside the build** (these are never covered by the manifest yes):

- A **prerequisite template that stores any input value** — name what is being stored before writing it, since the store is committed with the project (`safeguard.md` Rule 6).
- Any **drift resolution that overwrites or deletes** anything — always its own question, never folded into the manifest.

---

## Step 4 — Build in Dependency Order, With a Checkpoint Ledger

Follow the plan's Build Order, which always resolves to this type order:

1. **Env vars** — `manage_environment_variables`, under `negotiation/env-vars.md` §8 (the dashboard rejects artifacts referencing absent env vars, so these come first). **If APIs follow and the dashboard predates the new variables, restart it before step 3** (env-vars.md §6) — name that restart in the build manifest's build order.
2. **Saved queries** — `save_query`, then **smoke-test each immediately** with `test_query`.
3. **API definitions** — `create_or_update_api`. Adding test cases to an API the plan marks `reuse` is **additive** — never touch its other test cases.
4. **Prerequisite templates** — `manage_prerequisite_template(action="create")`, after the APIs they reference exist.
5. **Flows** — `validate_flow` first, then `create_flow`, one call per flow, subflows before parents.
6. **Test group** — `create_test_group` with the plan's items and order.

Pass each block's `args` **exactly as the plan holds them**, with only `${SECRET:…}` substituted. Do not reformat payloads, rename artifacts, add assertions, or "improve" a test case in passing.

Maintain a **checkpoint ledger** — every block `id` as ✅ built / ⏸ pending / ❌ failed — and report it whenever the build pauses.

### On failure — checkpoint, never silent rollback

If any call fails (including a failed `test_query` smoke test or a `validate_flow` error):

1. **Stop.** Build nothing downstream of the failure.
2. **Report** which block failed, the exact error, and the ✅/⏸/❌ ledger.
3. **Wait** for the user's decision: fix and retry that block / skip it and continue / abort.
4. **Never auto-delete** what was already created — deletes are safeguard-gated and the artifacts are usually still wanted.
5. On a go-ahead, **resume from the failed block**, not from the start.

Diagnose a failure at its root per `safeguard.md` Rule 2, and never work around a generator problem by editing generated files (Rules 1 and 3). Common structural causes are tabulated in `reference/build-workflow.md`.

---

## Step 5 — Report, Then Never Auto-Run

Report what was actually built:

- Every artifact created, by type and name, with its validation result.
- Every artifact reused.
- Anything skipped or failed, and why.
- The **coverage summary carried over from the plan** — including the not-automatable items with their reasons, so the user sees what the spec asked for that this build does not cover.
- Whether the plan file was amended during execution (drift resolutions, answered Open Questions).

Then ask — **as its own message**:

```
Built: 4 APIs, 4 flows, 1 test group. Would you like to run the test group now?
```

**Never auto-run.** A run is real traffic against a real environment: it needs its own explicit confirmation, naming the target and the environment (`safeguard.md` Rule 6). Running through the Docker runner or CI is `operations/docker-run.md`'s territory — hand off there rather than improvising a run.

---

## Step 6 — Mark the Plan Executed

After a successful build, append an **Execution Log** section to the plan file: date, environment, what was built, what was skipped, and the checkpoint ledger's final state. Re-executing a plan then starts from an honest record rather than rebuilding blindly — and a plan whose blocks all exist is a no-op with a report, not a duplicate build.

---

## What This Rule Is Not

- It is not a negotiator. It asks one confirmation (plus the explicit exceptions above), not a question per artifact — the plan already carries the user's decisions.
- It is not a planner. It never invents artifacts, scenarios, assertions, or wiring the plan does not contain; a gap in the plan is reported, not filled.
- It is not a runner. It builds; running belongs to an explicit follow-up under `safeguard.md` Rule 6 or `operations/docker-run.md`.
- It does not overwrite. A name collision found at pre-flight is a decision for the user, never an update in disguise.
- It does not edit generated files to make a build succeed (`safeguard.md` Rules 1 and 3).

> **On any conflict between this file and `api.md` / `query.md` / `flow.md` / `test-group.md` / `safeguard.md`, those rules win for the concern they own.** This file owns only the plan → artifacts execution orchestration.
