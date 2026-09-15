# QA Flow — Environment Variables Rule

## Scope

This file is the **single source of truth for environment variables** in QA Flow. It applies to **any AI assistant** in two ways:

1. **It gates the environment write tools** — `manage_environment_variables`, `create_environment`, `update_environment`, `delete_environment`, `set_environment_db_connection`. A request to add, change, import, delete, or reveal a variable, or to create / rename / clone / delete an environment or set its database connection, follows *Standalone Environment Writes* below.
2. **Every other rule defers to it** whenever it touches an env var — deciding whether a literal becomes one, writing a `{{env.*}}` / `{{context.env.*}}` / `env.*` template, handling a secret, checking a variable exists, or creating one as part of a larger build. Those rules keep **when** they ask (their own question cadence); this file owns **what** is true and **what** is allowed.

Container / CI `.env` operations — `QA_FLOW_ENV_FILE`, compose `env_file`, `REPORT_CHANNELS`, appending to `.env` on a host — belong to `operations/docker-run.md`. Choosing the environment for a **run** belongs to `safeguard.md` Rule 6 and `operations/docker-run.md`.

---

## 1. How Environments Are Stored

| Environment | Backing file | Id |
|---|---|---|
| **Default** | `.env` at the project root | `00000000-0000-0000-0000-000000000000` |
| Named (e.g. `live`, `staging`) | `.env.<slug>` (slug lowercase) | a UUID from `list_environments` |

- **The file is the source of truth.** The dashboard watches `.env*` files and reloads a variable the moment its file changes; `category` and `sensitive` are dashboard metadata layered on top.
- **A variable read from a file is created `sensitive: false`** — the `.env` format has no per-variable metadata. The flag is only reliable for variables written through the dashboard / MCP with it set.
- `ENV_FILES_DIR` relocates the `.env*` files (containers use it); the rules above do not change.

---

## 2. Template Syntax — Which Form, Where

Variable names are uppercase: `[A-Z_][A-Z0-9_]*`. A form that does not match its location **is not resolved** — it reaches the request as literal text.

| Where the reference lives | Form | Notes |
|---|---|---|
| API definition — the cURL (endpoint, headers, body) and test-case `payload` / `params` | `{{env.VAR}}` · `{{env.VAR\|default}}` | Resolved when the API is **created** (see §6) and again at run time |
| Flow step `payload`, `params`, `query_params`; `query` step inputs; `wait_until` / `conditional` visual fields | `{{context.env.VAR}}` | The `context.` prefix is required inside flows |
| Flow step `header_import[].variable` | `env.VAR` | Short form — no braces, no `context.` |
| Saved query values (SQL / Mongo) | `{{env.VAR}}` · `{{env.VAR\|default}}` | Resolved by `test_query` / at run time for the `environment_id` in play — **not** at definition time |
| Any `{{env.*}}` pinned to one environment | `{{env.<slug>.VAR}}` | Always reads `.env.<slug>`, whatever environment is active |
| `python_code` in `wait_until` / `conditional` | `get_env("VAR", default=None)` | |
| Build-plan `args` (`reference/build-plan-format.md`) | `${SECRET:VAR}` | Plan files only — resolved by the executor, **never** sent to a tool as-is |

`{{env.VAR|default}}` resolves to `default` when `VAR` is unset — a way to keep a value optional, never a way to hide a secret.

---

## 3. Where a Value Lives — Env Var, Flow Input, or Literal

Whenever a value would otherwise be **hardcoded** — a payload field, query param, header, query input, sub-flow input, condition comparison, test-group parameter — do not bake it in silently. Classify it and ask where it lives:

| Kind of value | Examples | Recommend | Reference |
|---|---|---|---|
| **Environment-specific / shared config** | base URLs and hosts, client / tenant / status IDs, feature flags | **env var** | `{{env.VAR}}` in an API, `{{context.env.VAR}}` in a flow |
| **Secret** (§5) | tokens, passwords, API keys, `Authorization` values, DB credentials | **env var**, `sensitive: true` | same as above — or a value exported by a login step (`header_import`) |
| **Run-varying test data** | user IDs, GUIDs, phone numbers, amounts, emails | **flow input** | `{{context.flow_input.X}}` (`reference/flow-structure.md`) |
| **Genuinely constant everywhere** | an API version segment, a fixed enum | literal — **only after the user confirms** | — |

Rules that always hold:

- **Base URLs are always env vars.** The main application uses `{{env.BASE_URL}}`; a secondary service gets its own variable (`{{env.HTTP_BIN}}`, `{{env.ESIM_BASE_URL}}`) — never re-point `BASE_URL` for another service.
- **Never copy an inline credential.** A cURL's `Authorization: Bearer eyJ…`, API key, or password becomes `{{env.VAR}}` or a login step's `header_import` — the literal never enters an artifact, a plan, or a message.
- **Test groups do not hold env-shaped values.** A group's `config.parameters` feed flow inputs; an environment-specific or secret value belongs in the flow as `{{context.env.VAR}}`, not pinned per group.
- **Placement is the user's call.** Several pending literals are presented together in **one** table-style message, each row picking its home (`reference/selection-format.md`, *Grouped low-priority parameters*); a single literal is one question. The calling rule decides *when* that question is asked.

---

## 4. Naming a New Variable

- **Reuse before creating.** If the target environment already holds a variable with the same value and purpose, propose reusing it instead of a duplicate under a new name.
- `UPPER_SNAKE_CASE`, named for what the value **is**: `TENANT`, `CLIENT_ID`, `ESIM_BASE_URL`.
- A secondary host is prefixed with its service (`ESIM_BASE_URL`, `KEYCLOAK_URL`).
- An API-key header maps to a variable named after the header: `Access-Token` → `ACCESS_TOKEN`, `Internal-Token` → `INTERNAL_TOKEN`.
- Typo-duplicates in a source (`tenant` / `teenant`) collapse into **one** variable when the user agrees.
- **Never rename an existing variable** as a side effect — every artifact referencing the old name would silently stop resolving.

---

## 5. Secrets

**What counts as a secret** — the same test the dashboard applies when redacting report snapshots:

- a name containing `PASSWORD`, `PASSWD`, `SECRET`, `API_KEY` / `APIKEY`, `PRIVATE_KEY`, `CREDENTIAL`, or `WEBHOOK`;
- a name containing `TOKEN` — **unless** its last segment is structural (`DB_TOKEN_HOST`, `DB_TOKEN_PORT`, `DB_TOKEN_USER`, … are connection metadata, not credentials);
- any `Authorization` header value, and any value that embeds credentials in a URI (`mongodb://user:password@host`) whatever the variable is called.

**How a secret value gets in:**

1. **Ask for the value in the conversation** — one grouped question naming every secret still needed. **Never echo the value back**; refer to it by variable name and show it as `•••` in every summary, confirmation, and report.
2. **Write it with `sensitive: true`** — on `add`, and on `update` when an existing variable is found to hold a secret.
3. **If the user defers** ("I'll set it later"), create it with the value `REPLACE_ME` and `sensitive: true`, and state that anything using it will fail until the real value is set in the dashboard or the environment file.
4. **Never write a secret value into a file** — not a build plan (`${SECRET:VAR}` only), not a prerequisite template, not a test case, not a steps document or report.
5. **Never invent one.** A missing token, password, or key is asked for — never guessed, never copied from docs, examples, or another environment.

**Existing secrets stored as non-sensitive.** Variables loaded from a file start `sensitive: false` (§1), so a real password can sit unmasked. Whenever you read an environment and a variable matches the secret test above while `sensitive` is `false`, **name it and offer** to mark it sensitive (`manage_environment_variables` `update`, `sensitive: true`) — as its own question. **Never flip the flag without that yes**, and never print its value while pointing it out.

**Revealing values.** `manage_environment_variables(action="export", include_sensitive=true)` returns secrets in clear text. Use it only when the user explicitly asks for the values, confirm first, and still never repeat a secret value in chat.

---

## 6. Existence, Creation Order, and the Dashboard Process

### Checking a variable exists

- **A listing proves a variable EXISTS, never that it is ABSENT.** `get_environment` output is size-capped: when it reports `truncated: true` (`showing` < `total`), a variable missing from it may simply have been cut — the same trap as `reference/name-availability.md`.
- **Confirm absence** with `manage_environment_variables(action="export")` (read-only; sensitive values masked), or — when the project is local — by reading the **names** in the backing file (`.env` / `.env.<slug>`) without printing any value. Only then treat a variable as missing.
- These existence checks are read-only and are always allowed where a rule permits an environment read.

### Create before use

**Every missing variable is created before the first artifact that references it.** The dashboard rejects `create_or_update_api` when a `{{env.VAR}}` in its cURL or test cases does not resolve.

### Which environment an API definition is resolved against

`create_or_update_api` resolves `{{env.VAR}}` against the **dashboard's own process environment** — the Default `.env` as loaded when the dashboard started — **not** against the environment the tests will later run in. So:

- A variable an **API definition** references must exist in the **Default** environment (`.env`), even when runs target a named environment. When the real value differs per environment, create it in Default *and* in each named environment that needs its own value.
- Flows, saved queries, and runs resolve at **run time** against the active environment (`os.environ` → `.env` → `.env.<slug>` → the environment's DB connection), so they need the variable only in the environment they run in.

### The restart step

A dashboard started **before** a variable was added does not see it: its env loader serves the process environment captured at start-up and does not re-read `.env`. The symptom is:

```
HTTP 500 … Environment variable 'VAR' not found. Available variables: …
```

— raised before any file is written, so a retry is clean. Therefore:

- **After creating a variable that an API definition will reference, restart a long-running dashboard before the first `create_or_update_api` that uses it.** Name the process (pid / port, or the container) in a confirmation — restarting stops a running service. Then verify `health_check` and continue.
- **Only API creation needs it.** `save_flow` and suite generation merely warn about an unresolved variable; runs pick up new values without a restart.
- In a build that creates variables and then APIs, this restart sits **between** those two blocks in the build order and is named in the build confirmation.
- Seeing that exact error for a variable you just created is this issue — not a defect in the plan or the API.

---

## 7. Standalone Environment Writes (the Gated Tools)

When the user's request is itself an environment write, negotiate before writing:

- **No write before negotiation.** The only calls allowed before the first question are `list_environments` and `get_environment` on the environment the request names, plus the §6 existence confirmation — nothing else.
- **One topic per message; the target environment is always its own question** unless the request named it unambiguously (picker from `list_environments`, per `reference/selection-format.md`).
- **Confirm before writing**, naming the environment, the variable(s), each value (`•••` for secrets), the `sensitive` flag, and the `category`. **Never auto-chain** another action afterwards.

Per tool:

| Action | Grouped question (one message) | Extra checks before the write |
|---|---|---|
| `manage_environment_variables` **add** | name (§4) + value (§5 for secrets) + `sensitive` + `category` (default `Custom`) | §6 existence confirmation — an existing name is an **update**, never a silent overwrite |
| **update** | which fields change (value / `sensitive` / `category`) | show the current `category` / `sensitive` and what changes; a secret's value stays masked |
| **delete** | confirm the exact variable | **search for references first** — `{{env.VAR}}`, `{{context.env.VAR}}`, `env.VAR`, `get_env("VAR")` across API definitions, flows, and saved queries — and name every dependent; deleting breaks them at their next run or regeneration |
| **import** | the variable set, previewed as a table with secrets masked | a name that already exists returns **409** — list the conflicts; resolve each as skip / rename / explicit update, never a blanket overwrite |
| **export** | — (read) | `include_sensitive=true` only per §5 *Revealing values* |
| `create_environment` | name + description; or `clone_from_env_id` | a clone copies **every** variable, secrets included — say so |
| `update_environment` | new name / description | if the rename changes the environment's slug, anything pinning `{{env.<old-slug>.VAR}}` stops resolving — search for such references first |
| `delete_environment` | confirm the exact environment | **destructive** — deletes the environment and all its variables; state the variable count, never delete **Default**, and name any group / plan / CI job known to run against it |
| `set_environment_db_connection` | `db_type`, `host`, `port` (required), `user`, `database_name` + the password (§5, masked) | `remove=true` deletes the override — confirm; a password is a secret |

Report the real result — what was created / updated / deleted, masked values, any restart still owed — and ask before anything that depends on it.

---

## 8. Env Writes Inside Another Workflow

Collection import, flow import, plan execution, and any conversational build that needs a variable create it **as part of their own build**:

- **The parent workflow's single build confirmation authorizes those writes** — no separate per-variable confirmation. That confirmation must list every variable being written: environment, name, value (`•••` for secrets), `sensitive`, and whether it is new.
- **Every check in this file still applies inside the build**: §6 existence confirmation, §5 secret collection and masking, `sensitive: true` for secrets, create before use, the Default-environment requirement for API references, and the restart step before the first API that needs a new variable.
- Variables come **first** in the build order; a failed variable write stops the build at the parent rule's checkpoint like any other block.
- **Planning never writes.** `negotiation/spec-plan.md` records variables in the plan's Environment Variables section; `operations/plan-execute.md` writes them.

---

## What This Rule Is Not

- Not the run-environment gate — choosing where tests run is `safeguard.md` Rule 6 / `operations/docker-run.md`.
- Not container or CI configuration — `.env` files on a host, compose `env_file`, `QA_FLOW_ENV_FILE`, and `REPORT_CHANNELS` are `operations/docker-run.md`.
- Not a reason to skip the calling rule's cadence — an API, flow, query, or import still asks its env-var question where its own rule puts it; this file decides the options, the recommendation, and what must be true before writing.

> **On any conflict between this file and another rule about environment variables, this file wins.** For everything else, the owning rule wins.
