---
name: qa-flow-global-values
description: MUST be used whenever a QA Flow task touches the Global Values store — shared named values referenced as {{global.KEY}} (typically an access token produced by a prerequisite template and cached for a TTL) — directly ("store the access token globally", "make all APIs use the shared token", "add a global value", "refresh / clear the global token", "opt this API into the global Authorization header", "show the global values") via manage_global_value / set_api_global_injections, AND whenever another qa-flow skill would hardcode an auth header, add a login step, or attach a prerequisite for a target API while a global value with a matching inject rule exists. Enforces one-question-per-message negotiation (key → source: producer template or hand-set value → TTL/sensitive/inject settings → API opt-ins), the one-namespace caveat (one cached value for every environment — propose clear before switching), secrets never echoed and kept sensitive, explicit yes before delete / refresh / clear / reveal, and never auto-running a producer to test. Does NOT apply to environment variables ({{env.*}} — qa-flow-env-vars) or to running an API with an ad-hoc prerequisite chain (qa-flow-write-safeguard).
---

# QA Flow — Global Values

Global values are governed by one rule file. It is the source of truth — this skill exists only to make sure it is loaded and followed. **On any conflict between this summary and the rule file, the rule file wins**; keep this summary in sync when the rule changes.

## Do this first — read the authoritative rule

- **Global values** → [ai-rules/negotiation/global-values.md](ai-rules/negotiation/global-values.md) — the key shape, reference syntax, negotiation, secrets, use inside other workflows.
- **Secrets** → [ai-rules/negotiation/env-vars.md](ai-rules/negotiation/env-vars.md) §5.
- **Any QA Flow MCP write** → also [ai-rules/safeguard.md](ai-rules/safeguard.md).

Present every choice per [ai-rules/reference/selection-format.md](ai-rules/reference/selection-format.md).

## The non-negotiable constraints (full detail is in the file above)

1. **No tool calls before the first question** except one `list_global_values` existence/name check.
2. **One topic per message:** key → source (producer template, or hand-set value) → settings (TTL + sensitive + inject header/prefix, grouped) → API opt-ins (one multi-select). Then one confirmation summary.
3. **One namespace, one cached value for every environment.** Say so when it matters and propose `clear` before an environment switch.
4. **Secrets:** a hand-set token is collected in chat, never echoed, kept `sensitive: true`; a producer is preferred because it renews itself.
5. **Explicit yes** before `delete` (after listing references and opt-ins), `refresh` (it runs the producer), `clear`, and `reveal`.
6. **Never auto-run** a producer or an API to "test" the key.
7. **Syntax:** `{{global.KEY}}` in payloads / params / query inputs / sub-flow + flow inputs / ui scripts; `global.KEY` in `header_import`. Prefer opting an API in (`set_api_global_injections`) over rewriting its headers.

## Session resilience (any assistant)

- If you can no longer see the rule file's full text, or which answers were agreed, **re-read `global-values.md`** and restate them before writing (safeguard Rule 7).
- If the client has a native single-question picker (e.g. AskUserQuestion), prefer it — still one question per message.
