---
name: qa-flow-env-vars
description: MUST be used whenever a QA Flow task touches environment variables or environments — directly ("add an env var", "set the tenant for staging", "change BASE_URL", "import these variables", "delete this variable", "create/clone/rename/delete an environment", "set the staging DB connection", "show me the env values") via manage_environment_variables / create_environment / update_environment / delete_environment / set_environment_db_connection, AND as the shared reference every other qa-flow skill loads whenever it decides whether a literal becomes an env var, writes a {{env.*}} / {{context.env.*}} / env.* template, handles a secret (token, password, API key), checks a variable exists, or creates variables as part of a build (collection import, flow import, plan execution). Enforces the syntax-per-location table, env var vs flow input vs literal placement, secrets collected in chat but never echoed and written sensitive:true, existing unmasked secrets flagged (never silently flipped), existence confirmed beyond truncated listings, create-before-use with API references resolved against the Default .env, the confirmed dashboard restart before the first create_or_update_api that needs a new variable, reference checks before deletes, and negotiate-then-confirm for standalone environment writes. Does NOT apply to container/CI .env handling, REPORT_CHANNELS or QA_FLOW_ENV_FILE (qa-flow-docker-run), or to choosing the environment for a run (qa-flow-write-safeguard / qa-flow-docker-run).
---

# QA Flow — Environment Variables

Environment variables are governed by one rule file. It is the source of truth — this skill exists only to make sure it is loaded and followed. **On any conflict between this summary and the rule file, the rule file wins**; keep this summary in sync when the rule changes.

## Do this first — read the authoritative rule

- **Environment variables** → [ai-rules/negotiation/env-vars.md](ai-rules/negotiation/env-vars.md) — syntax, placement, naming, secrets, existence, create-before-use, the restart step, standalone environment writes, and writes inside other workflows.
- **Any QA Flow MCP write** → also [ai-rules/safeguard.md](ai-rules/safeguard.md).

Present every choice per [ai-rules/reference/selection-format.md](ai-rules/reference/selection-format.md).

**Loaded by another skill?** Follow that skill's question cadence for *when* to ask, and this rule for the options, the recommendation, and what must be true before writing.

## The non-negotiable constraints (full detail is in the file above)

1. **Right form for the location.** `{{env.VAR}}` in API definitions and saved queries; `{{context.env.VAR}}` in flow payloads / params / query inputs / conditions; `env.VAR` in `header_import.variable`; `{{env.<slug>.VAR}}` to pin one environment; `get_env()` in `python_code`; `${SECRET:VAR}` only inside build-plan files. A mismatched form is sent as literal text.
2. **Never hardcode silently.** Config and secrets → env var; run-varying test data → flow input; a literal only after the user confirms. Base URLs are always env vars (a secondary service gets its own). Several literals → one table-style question.
3. **Secrets:** ask for the value in the conversation, **never echo it** (`•••` everywhere), write it `sensitive: true`; if deferred, a sensitive `REPLACE_ME` placeholder with the consequence stated. Never write a secret into any file, never invent or copy one.
4. **Flag unmasked secrets you see** — a secret-looking variable with `sensitive: false` is named and offered a fix as its own question; never flipped without a yes, never printed.
5. **A listing proves a variable exists, never that it is absent.** `get_environment` truncates — confirm absence with `export` or by reading the backing file's names.
6. **Create before use — in the Default `.env` for API references.** `create_or_update_api` resolves `{{env.VAR}}` against the dashboard's start-up environment, not the run environment.
7. **Restart before the first API that needs a new variable.** A long-running dashboard does not see variables added after start-up (`HTTP 500 … Environment variable 'VAR' not found`, no files written). Confirm the restart naming the process, verify `health_check`, then continue. Flows, queries and runs need no restart.
8. **Standalone writes negotiate first:** only `list_environments` / `get_environment` / existence checks before the first question; the target environment is its own question; confirm environment + variables + masked values + flags before writing. Deletes search for every reference first; `delete_environment` is destructive and never targets Default; `import` conflicts (409) are resolved per name, never blanket-overwritten.
9. **Inside another workflow's build**, that workflow's one confirmation authorizes the env writes — but it must list them, and every check above still applies.

If you find yourself about to call an environment write tool before those checks and the confirmation, stop and ask instead.

## When this skill does NOT apply

- Host / container / CI `.env` files, compose `env_file`, `QA_FLOW_ENV_FILE`, `REPORT_CHANNELS` → **qa-flow-docker-run**.
- Picking the environment a test run uses → **qa-flow-write-safeguard** (MCP runs) or **qa-flow-docker-run** (runner / CI).
- Information-only questions about syntax ("which form goes in a header_import?") → answer from the rule; nothing is written.

## Session resilience (any assistant)

- If you can no longer see the rule file's full text, or which variables and values (masked) were agreed, **re-read `env-vars.md`** and restate them before writing (safeguard Rule 7).
- If the client has a native single-question picker (e.g. AskUserQuestion), prefer it for the environment choice, placements, and confirmations — still one question per message.
