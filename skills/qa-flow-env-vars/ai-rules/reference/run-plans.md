# Reference — Run Plans, Channels, Exit Codes, Failure Triage

Companion tables for [operations/docker-run.md](../operations/docker-run.md). Read this when authoring a plan file, choosing channels, or diagnosing a failed run.

---

## Plan format

YAML is **configuration only**; Python defines the structure. Keep the pair side by side in a `plans/` folder — a relative `plan:` resolves next to the YAML.

```yaml
# plans/nightly.yml — configuration
channels: [email, teams]          # optional — overrides REPORT_CHANNELS
envInputs:                        # optional — plan-wide flow inputs
  client_id: "64b94b80f1a790cc22e12f53"
plan:
  plan: nightly_plan.py           # REQUIRED — the Python file holding the tree
```

```python
# plans/nightly_plan.py — structure (defines `plan`, does NOT call run_plan)
from docker.run_plan import Sequence, Parallel, Flow, Group, LoadTest

plan = Sequence(
    Parallel(Flow("login_flow"), Flow("csrf_demo")),
    Group("regression_suite"),
    Parallel(Flow("checkout"), LoadTest("checkout", users=20)),
)
```

A standalone script the user executes directly is the *other* shape — it imports `run_plan` as well and ends with `sys.exit(run_plan(plan, channels=[...]))`. Never mix the two: a file referenced by a wrapper's `plan:` key must not call `run_plan()`.

| Python node | Meaning | Example |
|---|---|---|
| `Sequence(a, b, …)` | Children run in order | `Sequence(Flow("a"), Flow("b"))` |
| `Parallel(a, b, …)` | Children run at once | `Parallel(Flow("a"), Flow("b"))` |
| `Flow("name")` | A flow, API or suite test | `Flow("login_flow", inputs={...})` |
| `Group("name")` | A test group (one unit; its items stay ordered) | `Group("regression")` |
| `LoadTest("name")` | A stress test | `LoadTest("checkout", users=50, spawn_rate=5, duration="5m")` |

**Options:** `stop_on_failure=True` on `Sequence` or `Group`; `max_workers=N` on `Parallel`; `inputs={...}` on any leaf.

**YAML keys:** `plan` (required), `channels`, `envInputs` (legacy alias: `inputs`), `env_file`, `max_workers`.

**Input cascade — highest wins:** the leaf's own `inputs` → the plan's `envInputs` → a `FLOW_INPUT_<NAME>` environment variable → the flow's own non-empty `default_value`.

**Failure semantics:** a failing leaf never stops the plan by default. Opt in per branch with `stop_on_failure=True`; skipped branches appear in the report as **skipped**, never silently dropped.

**Validation:** `--dry-run` validates and runs nothing. Every artifact must resolve and every `required` input must have a value; all problems are reported at once with exit 4.

---

## Environments

| Mechanism | How | Scope |
|---|---|---|
| `QA_FLOW_ENV_FILE=.env.qa` | Swaps which host `.env` compose injects at container start | That one `docker compose run` invocation |
| UI Environment selector | Overlays the named environment (stored in the `qa_storage` volume) per execution | Dashboard- and MCP-triggered runs |

Use `QA_FLOW_ENV_FILE` for CLI/CI runs and the UI selector for dashboard-triggered runs. Don't pass `env_file` to `run_plan()` in container context — compose already owns the environment there; that parameter is for local CLI use. Every background job stores a snapshot of the environment it used, so which backend a run targeted is always provable.

---

## Channels

Every enabled channel fires on every run. A channel missing its credentials warns and skips — it never crashes the run.

| Channel | Needs | Sends |
|---|---|---|
| `email` | `SMTP_HOST/PORT/USER/PASSWORD`, `TEAM_EMAILS` | Full breakdown in the body; HTML report, JSON summary and per-step details attached |
| `teams` | `TEAMS_WEBHOOK_URL` | Compact Adaptive Card — per-step status, timings, failure reasons |
| `jira` | `JIRA_BASE_URL`, `JIRA_USER`, `JIRA_API_TOKEN`, `JIRA_PROJECT_KEY` | Creates or comments on one rolling bug; attaches the report |
| `jira-webhook` | `JIRA_WEBHOOK_URL` | Flat JSON to a Jira Automation trigger |
| `gitlab` | — | Publishes `junit.xml` into the pipeline UI |

Per-run overrides: `--channels email,teams`, `--no-email`, `--require-delivery`. Teams needs a Power Automate workflow URL ("When a Teams webhook request is received" → "Post card in a chat or channel") — the classic Office 365 connectors are deprecated.

## Exit codes

| Code | Meaning |
|---|---|
| `0` | Everything passed and delivered |
| `1` | A test, group or load test failed |
| `3` | Tests fine, a channel couldn't deliver |
| `4` | Plan didn't validate — nothing ran |

Delivery problems never mask a test failure: a failing run stays `1`.

---

## Volumes (standalone / named-volume setups)

| Volume | Holds |
|---|---|
| `qa_datasources` | Flows, API definitions, test groups, saved queries |
| `qa_testcases` | The generated pytest files — what actually runs |
| `qa_apis` | Generated API wrapper classes |
| `qa_userdata` | `global_data.py` — a path entry per artifact created |
| `qa_userqueries` | Saved SQL/NoSQL query methods — what a flow's DB step calls |
| `qa_configs` | `configs.yml` — data-source mapping per artifact |
| `qa_storage` | Environments, jobs, load-test queue |
| `qa_assets` | Uploaded test assets — what an API that posts a file reads |
| `qa_testsuites` | Group and load-test result records |
| `qa_reports` | HTML report, JSON summary, `junit.xml`, history |

`qa_userdata` and `qa_configs` look like housekeeping but are load-bearing — generating a flow appends a line to each. A named volume is not a bind mount: it lives in Docker's own storage, so on a standalone setup nothing generated is visible on the host (`docker compose cp` pulls files out). Scaffolded projects bind-mount instead, so artifacts land straight in the repo.

---

## Failure triage

| Symptom | Cause & fix |
|---|---|
| `no such service: run` | `--profile tests` missing, and it goes **before** the subcommand |
| `no configuration file provided: not found` | No compose file here — likely a cloned `demo-qa-automation`, which hosts the image and is not a project. Scaffold with `qa-flow init` |
| `pull access denied for qa-flow-dashboard` | The framework's compose file is in play (it builds from source, local tag). Run `qa-flow update` to restore the right one |
| `qa_run.sh: executable file not found` | Same cause — a stale locally-built image with none of the runner scripts |
| A script that plainly exists reports `no such file or directory` (`validate_container.sh`, `qa_run.sh`) while the dashboard stays `healthy` | CRLF line endings from a Windows checkout: the shebang reads as `/bin/sh\r`, and the error names the script, not the missing interpreter. The dashboard is unaffected because its CMD and healthcheck are Python. Confirm with `head -1 docker/scripts/validate_container.sh \| cat -A` (a trailing `^M`). Fix the checkout once — `git add --renormalize . && git checkout .` now that `.gitattributes` pins `*.sh` to LF — then rebuild; the Dockerfile also strips CR at build time |
| `[+] Building` in a user project | Wrong folder or wrong compose file; only the framework's own compose builds |
| Container `healthy` but the page won't load | The published port maps to a port nothing serves. In `host:container` the right side is always `5001` |
| `echo "X=Y" >> .env` had no effect | No trailing newline in older scaffolds — it glued onto the last line. Use `printf '\nX=Y\n' >> .env` |
| Registry pull "access denied" | PAT lacks `read_registry`, or `docker login registry.gitlab.com` was never run |
| Dashboard exits right after start | A configured database is unreachable. Fix connectivity or set `DB_TYPE=file_storage` |
| Tests import-error in the runner but work in the UI | `qa_userdata` / `qa_configs` aren't mounted on the `run` service |
| New image pulled, behaviour unchanged | The container is still the old one — `docker compose up -d --force-recreate dashboard` |
| `pull` says "up to date" though a fix merged days ago | `:latest` moves only on a `vX.Y.Z` tag, never on a merge — a release has to be cut |
| Two parallel runs overwrote each other's report | Separate invocations share `Reports/`. Give each its own `-e REPORTS_DIR=/app/Reports/<name>` |
| A channel silently did nothing | Its variables are unset — it warns and skips by design. `--require-delivery` makes that loud |
| `port is already allocated` on `up` | Another dashboard holds 5001 — set `QA_FLOW_PORT` |
| Compose edits vanished after `qa-flow update` | `docker-compose.yml` is framework-owned. Move changes to `docker-compose.override.yml`; the updater kept a backup |
| `--plan my.yml` → `FileNotFoundError` | The plan is on the host, not in the container. Mount its folder: `-v "$PWD/plans:/app/plans:ro"` |
| `❌ PLAN_FILE is not set` from a job that used to work | The retired `PLAN_TARGETS`/`PLAN_GROUPS`/`PLAN_MODE`/`PLAN_YAML` interface — point `PLAN_FILE` at a committed plan YAML |
| Plan job dies with `❌ Nothing to run` despite `PLAN_FILE` | The image predates the file-based interface — promote/pull a newer one |
| CI job green but the report URL 404s | The report was never copied out — the `set -e` / `PLAN_EXIT` pattern in operations/docker-run.md Rule 4 |
| Channels skip only on a feature branch | Protected CI/CD variables don't reach unprotected branches |
