---
name: qa-flow-plan-execute
description: MUST be used whenever the user asks to execute, build, or apply a QA Flow **build plan** — the `test-plans/<slug>.build-plan.md` artifact written by the qa-flow-spec-plan skill. Triggers on requests like "execute the plan", "build the plan", "apply test-plans/e2e-spec.build-plan.md", "go ahead and create everything in the plan", or "run the build plan I approved". Turns the plan's JSON blocks into real artifacts through the QA Flow MCP server — env vars, saved queries, API definitions, prerequisite templates, flows, and the test group — in the plan's dependency order, behind ONE build confirmation that names the counts, the targets and the environment. Verifies the plan first (parses every block, checks dependencies and build order, refuses literal secrets, unresolved placeholders, endpoints carrying a query string, dangling `{{context.*}}` references, and any non-empty Open Questions), then runs a read-only pre-flight to detect drift since the plan was written — a name that now exists is never overwritten, it becomes a user decision. Builds exactly what the plan says: never re-negotiates decisions the plan already settled, never invents or "improves" artifacts the plan does not contain. A failure (including a failed query smoke test or validate_flow) stops the build at a reported ✅/⏸/❌ checkpoint with nothing auto-deleted and nothing built downstream; a successful build reports coverage (including what the spec asked for that is not automatable) and asks — as its own message — before running anything. Does NOT apply to writing the plan (use qa-flow-spec-plan), to a Docker/CI **run plan** of already-built artifacts (use qa-flow-docker-run), or to building artifacts negotiated conversationally without a plan file (use the qa-flow-*-negotiation skills).
---

# QA Flow — Plan Execute (Build Plan → Built Artifacts)

Executing a build plan is governed by strict rules. This skill exists only to make sure they are loaded and followed. **On any conflict between this summary and the rule files, the rule files win**; keep this summary in sync when the rules change.

## Do this first — read the authoritative rules

Before doing anything else, Read these and follow them exactly:

- **The execution workflow** → [ai-rules/operations/plan-execute.md](ai-rules/operations/plan-execute.md) — the orchestration rule for this workflow.
- **The artifact it consumes** → [ai-rules/reference/build-plan-format.md](ai-rules/reference/build-plan-format.md) — sections, JSON block envelope, secrets, and the Validity Checklist this skill enforces.
- **Every write it performs** → [ai-rules/safeguard.md](ai-rules/safeguard.md).
- **Env vars and secrets** → [ai-rules/negotiation/env-vars.md](ai-rules/negotiation/env-vars.md) — existence checks, `sensitive: true`, which environment a variable must exist in, and the restart step.
- **What it builds** → [ai-rules/negotiation/api.md](ai-rules/negotiation/api.md), [ai-rules/negotiation/query.md](ai-rules/negotiation/query.md), [ai-rules/negotiation/flow.md](ai-rules/negotiation/flow.md), [ai-rules/negotiation/test-group.md](ai-rules/negotiation/test-group.md) — structural conformance.
- **Build order and failure causes** → [ai-rules/reference/build-workflow.md](ai-rules/reference/build-workflow.md).

## The non-negotiable constraints (full detail is in the files above)

1. **The plan is the contract.** Build exactly what its JSON blocks say, in its Build Order. **Never re-negotiate** what the plan settled — the user approved it so they would not answer twice — and **never improvise**: no artifact, test case, assertion, or payload edit the plan does not contain. A wrong or incomplete plan is reported, not patched mid-build.
2. **Verify before anything else.** Run the plan format's Validity Checklist: every block parses, dependencies resolve, Build Order is a valid topological order, no literal secrets, no `<fill at build>` placeholders, no `?` in an endpoint, no dangling `{{context.*}}`. **A non-empty Open Questions section blocks the build** — answer it into the plan first. A failing check refuses the build and names the failure.
3. **Pre-flight is read-only** — `list_api_definitions`, `list_queries`, `list_flows`, `list_test_groups`, `list_prerequisite_templates`, `get_environment`/`list_environments`, `health_check`. Nothing else before the confirmation. **`list_*` truncates silently at 20k chars**, so confirm every `create` name and every `reuse` target with a targeted `get_*` lookup (404 = free) rather than trusting the listing.
4. **Drift is a user decision, never a unilateral one.** A `create` whose name now exists is **not** overwritten: offer skip-if-identical / new name / stop. Never pass an existing name to a create tool and call it an update.
5. **Secrets resolve at build time** (`env-vars.md` §5) — read `${SECRET:VAR}` from the environment, ask (one grouped question, values never echoed) for whatever is unset, write secrets `sensitive: true`, and never write a resolved value back into the plan file.
6. **ONE build confirmation**, its own message, naming the counts by type, the reused artifacts, every drift resolution, the masked secrets, the build order, the coverage summary, and the environment. One yes authorizes the whole chain. Two things still need their own extra confirmation: a prerequisite template that would **store an input value**, and any drift resolution that **overwrites or deletes**.
7. **Build in dependency order** — env vars (then a confirmed dashboard restart when APIs need a variable the running dashboard predates — `env-vars.md` §6) → saved queries (each smoke-tested with `test_query`) → APIs (`create_or_update_api`; additions to a reused API are additive) → prerequisite templates (`manage_prerequisite_template`) → flows (`validate_flow` then `create_flow`, subflows before parents) → test group. Pass each block's `args` verbatim, with only `${SECRET:…}` substituted.
8. **Failure = checkpoint, never silent rollback.** Stop, report the block, the exact error and the ✅/⏸/❌ ledger, build nothing downstream, auto-delete nothing, and resume from the failed block on the user's go-ahead. Diagnose at the root (safeguard Rule 2) and never edit generated files to make a build pass (Rules 1 and 3).
9. **Report honestly, then never auto-run.** Report what was built, reused, skipped and failed, carry over the plan's coverage summary including the not-automatable items, then ask — as its own message — whether to run anything. A run is real traffic and needs its own confirmation naming target and environment; runner/CI execution belongs to **qa-flow-docker-run**.
10. **Record the execution** — append an Execution Log section to the plan file (date, environment, built/skipped, final ledger) so a re-execution starts from an honest record instead of rebuilding blindly.

## When this skill does NOT apply

- Writing or amending the plan → **qa-flow-spec-plan**.
- A Docker/CI **run plan** orchestrating already-built artifacts → **qa-flow-docker-run** (a build plan is never passed to `user_runner.py`).
- Building something negotiated conversationally, with no plan file → the **qa-flow-\*-negotiation** skills.
- Running, deleting, or duplicating existing artifacts → **qa-flow-write-safeguard**.

## Session resilience (any assistant)

- A long build can outlive the context window. If you can no longer see the rule file's full text or the checkpoint ledger, **re-read `plan-execute.md`** and restate the ✅/⏸/❌ ledger before continuing (safeguard Rule 7).
- If the client has a native single-question picker (e.g. AskUserQuestion), prefer it for the build confirmation, drift resolutions, and the run question — still one question per message.
