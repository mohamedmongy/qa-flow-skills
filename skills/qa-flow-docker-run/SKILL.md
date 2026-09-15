---
name: qa-flow-docker-run
description: MUST be used whenever the user asks to set up a QA Flow project, run their tests/plans, or wire the pipeline around them — "set up / scaffold my QA Flow project", "configure .env or docker-compose", "run my flow / group / load test", "run everything", "run them in parallel", "write me a run plan", "run the plan in the container", "add the QA Flow job to my .gitlab-ci.yml", "why did my run deliver nothing", "the report didn't arrive", "no such service: run", "build and publish the image", "cut a release tag" — i.e. the operational layer around the artifacts (project scaffolding, .env and compose, user_runner / qa_run.sh invocations, plan files, report channels, GitLab CI job, image build & tag-driven publishing). Enforces an execution gate: inspecting, validating (--dry-run) and authoring files are free, but ANY real run, docker build, image push, release tag or pipeline trigger needs one explicit confirmation that names the target, the resolved environment and the channels that will fire; destructive actions (docker compose down -v, stopping every container) state their blast radius in the question. Does NOT apply to authoring artifacts through the MCP server — creating/updating flows, APIs, test groups, queries, load tests or report-dashboard writes are owned by the qa-flow-*-negotiation skills and qa-flow-write-safeguard.
---

# QA Flow — Project Setup, Running Plans, and the Pipeline

The negotiation skills own *authoring* artifacts through the MCP server. This skill owns everything *around* them: creating and configuring the project, executing flows / groups / load tests / run plans (locally, in containers, from CI), delivering the reports, and building and publishing the framework image. The rules are the source of truth — this skill exists only to make sure they are loaded and followed. **On any conflict between this summary and the rule files, the rule files win.**

## Do this first — read the authoritative rules

Before doing anything else, Read these and follow them exactly:

- **The operations rule** → [ai-rules/operations/docker-run.md](ai-rules/operations/docker-run.md) — the execution gate, orientation, setup, running, pipeline, publishing.
- **The tables it points to** → [ai-rules/reference/run-plans.md](ai-rules/reference/run-plans.md) — plan node types and YAML keys, the input cascade, environments, channels, exit codes, volumes, and the symptom→cause triage table.
- **If the task also touches MCP writes** → [ai-rules/safeguard.md](ai-rules/safeguard.md) and the relevant negotiation rule.

Present every choice per [ai-rules/reference/selection-format.md](ai-rules/reference/selection-format.md); prefer a native single-question picker (e.g. AskUserQuestion) and keep it one topic per message.

## The non-negotiable constraints (full detail is in the files above)

1. **The execution gate.** Inspecting (`docker compose config/ps/logs`, `qa-flow validate`, `--dry-run`) and authoring files (plan `.py`/`.yml`, `.env`, `docker-compose.override.yml`, the CI job) are free. **Any real run, `docker compose up/pull/run`, `docker build`, `qa-flow init`/`update`, or `docker login` takes ONE explicit confirmation.** Pushes, release tags, pipeline triggers, `docker compose down -v` and `docker stop $(docker ps -q)` are confirmed *loudly*, with the blast radius stated in the question itself.
2. **Name the environment and the channels in that confirmation.** A run resolves to a specific backend (`QA_FLOW_ENV_FILE` / dashboard environment / CI variables) and fires every channel in `REPORT_CHANNELS` — real mail, a Teams card, a Jira issue. Both go in the question, never discovered afterwards.
3. **Never invent a credential.** PATs, SMTP passwords, webhook URLs, DB passwords and tokens come from the user or their existing environment. Values that appear in the guide are placeholders — never copy one into a file or command, never echo a secret's value, refer to it by variable name.
4. **Orient before acting.** Framework repo (its compose *builds*; runs via `python docker/user_runner.py`), scaffolded project (pulls the published image; runs via `docker compose --profile tests run --rm run`), standalone folder (named volumes, nothing visible on the host), or nothing set up. The same command means different things in each.
5. **Never assume independence.** `--parallel` is the user's assertion, not your inference — recommend sequential for artifacts sharing state and for rate-limited backends, and express real ordering structurally (flow steps, group items).
6. **Plan files: two shapes, never mixed.** A script the user executes *calls* `run_plan()`; a Python file referenced by a YAML wrapper's `plan:` key only *defines* `plan`. Keep the YAML and its Python file side by side in `plans/`, and always `--dry-run` before proposing a run — validation reports every problem at once (exit 4) and nothing executes.
7. **`docker-compose.yml` is framework-owned.** User changes belong in `docker-compose.override.yml`; `qa-flow update` overwrites the former and never touches the latter.
8. **Publishing is a git tag, not a merge** (framework repo only). Always `--target` and `--pull` on builds; verify inside the image before publishing; the tag must match `vX.Y.Z` exactly; manual pushes go to `qa-flow-test-automation`, never to the promotion target.
9. **Diagnose from the triage table first.** Say which row you matched and apply that one fix — no stacked speculative changes.
10. **Never auto-run after building.** Writing the plan, the compose change or the CI job ends with a question, not with the run.

If you find yourself about to execute a confirm-bucket command before the user has said yes, stop and ask instead.

## When this skill does NOT apply

- Creating or updating flows, APIs, test groups, queries, load tests, or report-dashboard writes → the matching **qa-flow-\*-negotiation** skill.
- Deleting/duplicating artifacts, asset writes, or directly running an existing group/suite through MCP → **qa-flow-write-safeguard**.
- Environment variables and environments through MCP (`{{env.*}}` references, secrets, `manage_environment_variables` and the environment tools) → **qa-flow-env-vars**.
- Information-only requests ("what does `--parallel` do?", "show me the plan format") → answer from the rules; no confirmation needed because nothing executes.

## Session resilience (any assistant)

Setups and debugging sessions can outlive the context window. If you can no longer see the rule file's full text, or which environment and channels were confirmed, **re-read `operations/docker-run.md`** and restate the confirmed setup before continuing (safeguard Rule 7).
