# Operations Rule — Set Up a Project, Run Plans, Configure the Pipeline

This rule governs everything **around** the artifacts: creating and configuring a QA Flow project, running flows / APIs / suites / groups / load tests and run **plans** (locally, in containers, or from CI), wiring the report channels, and building/publishing the framework image. The negotiation rules own *authoring* artifacts through the MCP server; this rule owns *setting up and executing* them.

Source guide: `docs/qa-team-docker-onboarding.md` (long form, with the full compose file, appendices and cheat sheet). This rule is self-contained — read the guide only when the user asks for it or when you hit something this rule doesn't cover. Node types, channels, exit codes and the symptom→cause table live in [reference/run-plans.md](../reference/run-plans.md).

---

## Rule 0 — The execution gate (non-negotiable)

Actions are in one of three buckets. **Never move an action up a bucket to save a round trip.**

| Bucket | Examples | Requires |
|---|---|---|
| **Free** — inspect and author | `docker compose config/ps/logs`, `docker images`, reading `.env` *keys*, `qa-flow validate`, `--dry-run` plan validation, writing/editing plan files, `.env` edits, `docker-compose.override.yml`, a CI job in `.gitlab-ci.yml` | Nothing — just report what you did |
| **Confirm once** — real execution or state change | any actual test/group/load-test/plan run, `docker compose up/pull/run`, `docker build`, `qa-flow init` / `qa-flow update`, `docker login` | ONE explicit user confirmation naming target + environment |
| **Confirm loudly** — outward-facing or destructive | `docker push`, `git tag` + `git push` of a release tag, triggering a pipeline, `docker compose down -v`, `docker stop $(docker ps -q)`, anything touching a shared/production backend | Explicit confirmation that states the blast radius in the question itself |

Two things are never implicit:

1. **Which environment the run targets.** Before any real run, state the resolved backend — the env file (`QA_FLOW_ENV_FILE`), the dashboard environment, or the CI variables — and get it confirmed. A run against the wrong `BASE_URL` is the failure mode this rule exists to prevent.
2. **What the run will deliver.** Report channels fire on every run. If `REPORT_CHANNELS` includes `email`, `teams` or `jira`, say so in the confirmation — real people get mailed, a card gets posted, a Jira issue gets created or updated.

**Never auto-run after building.** Creating a plan file, a compose file or a CI job ends with a question — never with the run. This matches the negotiation rules: build, then ask.

**Never invent a credential.** SMTP passwords, PATs, webhook URLs, DB passwords and tokens come from the user or from their existing environment. The guide's samples are illustrative — treat every value in it as a placeholder, never copy one into a file or a command, and never echo a secret's value back into the conversation or into a commit. Refer to a secret by its variable name.

---

## Rule 1 — Orient before you act

The same commands mean different things in different folders. Establish which one you are in **first**, from the filesystem, not from assumption:

| Signal | You are in | Consequences |
|---|---|---|
| `cli_package/`, `docker/run_plan.py`, a `docker-compose.yml` containing `build:` | **The framework repo** (maintainer) | Runs go through `python docker/user_runner.py`. Its compose **builds from source** and tags `qa-flow-dashboard:latest` — an image in no registry. §2 publishing applies here and only here |
| `code_generator/`, `docker-compose.yml` whose services use `image: ${QA_FLOW_IMAGE:-…demo-qa-automation:latest}` (no `build:`) | **A scaffolded user project** | Runs go through `docker compose --profile tests run --rm run …`. Artifacts bind-mount into the repo |
| Just `docker-compose.yml` + `.env`, nothing else | **A standalone dashboard folder** | Same commands; artifacts live in named volumes, invisible on the host — pull them out with `docker compose cp` |
| No compose file at all | Nothing set up yet, or a cloned `demo-qa-automation` | `demo-qa-automation` **hosts the image, it is not a project** — never clone it, scaffold instead (Rule 2) |

`[+] Building` in a user project always means the wrong compose file is present — the framework's, copied by hand. Fix it with `qa-flow update`, don't work around it.

---

## Rule 2 — Project setup (guide §3)

Ask for what's missing **one topic per message**, then confirm once and execute the whole setup.

**Order:**

1. **Registry access** — the image is in a private GitLab registry. The user needs a PAT with the **`read_registry`** scope and one `docker login registry.gitlab.com`. Never run `docker login` with credentials pasted into the command; let the user log in, or run it interactively at their request.
2. **Scaffold** — `qa-flow init <name> --yes --db <mongodb|postgres|file_storage>` inside an activated venv (`uv venv && source .venv/bin/activate`, then `uv pip install --upgrade qa-flow-automation`). It writes **both** `docker-compose.yml` and `.env`. An existing project upgrades instead: `qa-flow update`, then `qa-flow validate`.
3. **`.env`** — nothing is baked into the image; every URL, credential and channel setting comes from here. Confirm each group with the user: base URLs, database, report channels. **Start with `DB_TYPE=file_storage`** unless the user needs real data checks — the container is then self-contained. A configured-but-unreachable database makes the dashboard exit at startup; that is the framework being strict, not a bug.
4. **Port** — `QA_FLOW_PORT` only if 5001 is taken. In `host:container` the right side is always `5001`.
5. **Bring it up** (confirm-once bucket): `docker compose pull && docker compose --profile tests pull`, `docker compose up -d`, wait for `healthy`, then `docker compose exec dashboard validate_container.sh`.

**Two edit rules that bite:**

- Append to `.env` with `printf '\nX=Y\n' >> .env` — never `echo`. Projects scaffolded before the fix have no trailing newline, so `echo` glues the variable onto the last line (`DB_TYPE=mongodbX=Y`) and loses both.
- `docker-compose.yml` is **framework-owned**: `qa-flow update` overwrites it. User changes go in `docker-compose.override.yml`, which Compose merges automatically and updates never touch. If the user asks you to edit the compose file directly, say this and put the change in the override instead.

---

## Rule 3 — Running plans (guide §1 and §4)

**Same runner, two invocations.** Everything else is identical:

| Framework repo | User project |
|---|---|
| `python docker/user_runner.py …` | `docker compose --profile tests run --rm run …` |

`--profile tests` is required and goes **before** the subcommand; without it you get `no such service: run`.

Targets: a bare name (a flow, API or suite — `login_flow`, never the generated path), `-m <marker>`, `--group <name>`, `--load-test <group> --users N`, `--run-release <tag>`, `--compare-releases <a> <b>`, or `--plan <file>`.

### Sequential vs parallel

Everything runnable is normalised into a **unit** (one test file, one group, one load test). **One invocation always produces one report**, delivered once; `--parallel` only decides whether units wait for each other (`--parallel` = one worker per unit capped at 8, `--parallel N` = at most N, `--parallel 1` = sequential).

You cannot infer independence — **the user asserts it**. Ask before adding `--parallel`, and recommend staying sequential for artifacts that share state (a login that seeds a token, a flow that creates the row the next one reads) and for rate-limited or single-tenant backends. Where order matters, express it structurally: steps inside a flow always run in flow order; a group is one unit and its items stay sequential even under `--parallel` (and honour `--stop-on-failure`). Mixed needs → put the ordered work in a group and run that group alongside the independent artifacts.

Guaranteed either way: the report is ordered by declaration, not completion; output is buffered per unit; a crashed unit costs one unit, not the run; unknown targets/groups/releases abort at plan time in both modes.

### Plan files

Reach for a plan when one level of `--parallel` isn't enough ("these two together, *then* that group, *then* these two"). **Structure lives in Python; the YAML carries configuration and points at it** — see [reference/run-plans.md](../reference/run-plans.md) for node types, YAML keys and the input cascade.

**The two shapes are not interchangeable — mixing them is the most common authoring bug:**

- A script the user executes themselves **calls** `run_plan(...)` and is run as `python my_plan.py`.
- A Python file referenced by a YAML wrapper's `plan:` key only **defines** `plan` and must **not** call `run_plan()` — the wrapper's settings drive the run.

Keep the YAML and the Python file it references **side by side in a `plans/` folder**; a relative `plan:` resolves next to the YAML, and both containers and CI copy/mount the whole folder.

**Always `--dry-run` first** (free bucket). Nothing runs until the entire plan validates — every artifact must resolve and every `required` flow input must have a value — and you get all problems at once with exit 4. Fix them all before asking to run.

In a container the plan files live on the host, so mount the folder — `--plan` alone gives `FileNotFoundError`:

```bash
docker compose --profile tests run --rm -v "$PWD/plans:/app/plans:ro" \
  --entrypoint qa_run.sh run --plan plans/nightly.yml --dry-run
```

### Reports and channels

`REPORT_CHANNELS` in `.env` is the standing default; override per run with `--channels a,b`, `--no-email`, or `--require-delivery` (exit 3 if a channel fails). A channel missing its variables warns and skips — it never crashes the run, which is exactly why a "nothing was delivered" complaint is usually an unset variable, not a bug. One invocation = one report: for separate reports, use separate invocations each with its own `-e REPORTS_DIR=/app/Reports/<name>`, or they overwrite each other's summary.

### Upgrading the image

`docker compose pull && docker compose --profile tests pull`, then `docker compose up -d --force-recreate dashboard`. Artifacts are untouched. `:latest` tracks the newest **release** and moves only on a `vX.Y.Z` tag — a pull that changes nothing usually means no release has been cut, not that the pull failed. **Never suggest `docker compose down -v` to "reset"**: named volumes are seeded from the image only on first mount, and `-v` deletes everything the user created. It is a confirm-loudly action with the data loss stated in the question.

---

## Rule 4 — The pipeline (guide §5)

The CI job runs the same runner inside the promoted image. **The plan file is the entire interface**: `PLAN_FILE` = repo-relative path to the plan YAML (e.g. `plans/nightly.yml`), optionally `PLAN_CHANNELS` to override the plan's `channels:` for one run. The job copies the plan's **folder** into the container, so the Python plan the wrapper references rides along — which is why `PLAN_FILE` must live in a subfolder, not at the repo root.

When authoring or reviewing that job, these four are correctness, not style:

1. **`docker start -a plan_run && PLAN_EXIT=0 || PLAN_EXIT=$?`** — never `; PLAN_EXIT=$?`. The runner uses `set -e`, so the semicolon form aborts the job the moment a test fails, *before* the report is copied out — losing the report on exactly the runs that need it.
2. **User artifacts come from the checkout**, `docker cp`-ed in: `code_generator/`, `user_data/`, `Configs/`, plus the plan folder. The published image ships no user content. Dropping `user_data`/`Configs` produces import errors that look like framework bugs.
3. **`REPORT_RESULT_URL` is ours, not a GitLab built-in** — computed from `CI_JOB_URL`. Set it and the Teams card gains an "📄 Open test result" button pointing at the job's HTML artifact; unset, the button simply doesn't render.
4. **Protected CI/CD variables don't reach unprotected branches.** If SMTP/Teams variables are marked *Protected*, a feature-branch job sees none of them and every channel skips. Check this before debugging the channels themselves.

Also carry over from the job's env handling: `--env-file` takes each line verbatim as `KEY=VALUE` and cannot represent a newline, so multi-line variables are dropped whole; runner internals (`CI_*`, `DOCKER_*`, `RUNNER_*`, …) are filtered out rather than handed to the container.

Cross-project registry access has a hard limit worth knowing before proposing a design: GitLab's `CI_JOB_TOKEN` only ever authenticates against **its own project's** registry. Publishing to another project therefore runs as a downstream pipeline with its own credentials — never as a direct cross-project push.

Editing `.gitlab-ci.yml` is free; **triggering a pipeline is confirm-loudly.**

---

## Rule 5 — Build and publish the image (guide §2 — framework repo only)

Only applies in the `qa-flow-test-automation` repo. Everything here is confirm-once at minimum, and every push or tag is confirm-loudly.

- **Always pass `--target`.** `test` is the last stage, so a bare `docker build` produces the test variant, not the consumer image. `runtime` → the framework without `tests/`, published as `:latest` / `:X.Y.Z`. `test` → runtime + the framework's own suites, published as `:latest-test` / `:X.Y.Z-test`.
- **Always `--pull`.** The base is the floating `python:3.10-slim` tag; a stale cached layer would silently keep an unpatched CPython. The build refuses to proceed below the security floor (3.9.22 / 3.10.17 / 3.11.12 / 3.12.9 / 3.13.2) — a stdlib CVE no dependency scan can see. On a cached build the check prints nothing; that is normal.
- **Verify before publishing**: the framework suites inside the image (`run_framework_tests.sh` on the `-test` tag), a running-container probe on a free host port with `DB_TYPE=file_storage` (`validate_container.sh`, after the container reports `healthy`), and `python scripts/audit_dependencies.py --image …`.
- **Publishing is a git tag, not a merge.** Merging to `main` builds and tests but publishes only an immutable `:<short-sha>` nobody pulls. Pushing `vX.Y.Z` builds both targets, runs the suites *inside* the freshly built image, publishes `:X.Y.Z`, `:X.Y.Z-test`, `:latest`, `:latest-test`, then triggers the downstream promotion to `demo-qa-automation`. Nothing reaches consumers until the tests pass in that exact image.
- **The tag must match `vX.Y.Z` exactly.** `v1.9`, `1.9.130` and `v1.9.130-rc1` match no publish rule and quietly build nothing.
- **A tag is a release.** Before proposing one, check with the user that the version bump, migration script and changelog entry are in place, and that the commit being tagged is the one they mean.
- **Manual pushes go to `qa-flow-test-automation`, never to `demo-qa-automation`** — the demo project is the promotion target, and pushing straight to it bypasses the gate that makes `:latest` trustworthy. Consumers pin a private build with `QA_FLOW_IMAGE=…:mybranch`.

---

## Rule 6 — Diagnose from the symptom table first

When something fails, match the symptom in [reference/run-plans.md](../reference/run-plans.md) before theorising. Most failures here are configuration, not code: a missing `--profile tests`, the framework's compose file in a user project, an unmounted volume, a port mapped to the wrong container port, an unset channel variable, or a `:latest` that hasn't moved because no release was tagged. Say which row you matched, apply that fix, and re-check — don't stack speculative changes.

If the user reports a run that "used to work", check what changed in that order: the image (`docker images`, was it re-pulled?), the env file in effect, then the artifacts. And per safeguard Rule 7: if this rule's text is no longer fully visible mid-task, re-read it and restate the confirmed setup before continuing.
