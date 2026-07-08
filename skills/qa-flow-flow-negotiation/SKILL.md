---
name: qa-flow-flow-negotiation
description: MUST be used whenever the user asks to create or update a QA Flow flow (create_flow/update_flow) or API definition (create_or_update_api) via the qa-flow MCP server — including natural-language requests like "build/automate a test", "chain these API calls", "add/change a step in a flow", "test this endpoint end-to-end", or "add this curl as an API". Enforces the project's negotiate-before-you-build rules — one topic per message (related low-priority settings grouped into one question), never call MCP tools before negotiation, get explicit confirmation before building. Does NOT apply to information-only requests (list/show/get a flow or API) or to deleting/duplicating flows and APIs (see qa-flow-write-safeguard).
---

# QA Flow — Flow & API Negotiation

Creating or updating QA Flow **flows** and **API definitions** in this repo is governed by strict negotiation rules. These rules are the source of truth — this skill exists only to make sure they are loaded and followed. **On any conflict between this summary and the rule files, the rule files win**; keep this summary in sync when the rules change.

## Do this first — read the authoritative rule

Before doing anything else, Read the matching file and follow it exactly:

- **Flow** (`create_flow` / `update_flow`) → [ai-rules/negotiation/flow.md](ai-rules/negotiation/flow.md)
- **API definition** (`create_or_update_api`) → [ai-rules/negotiation/api.md](ai-rules/negotiation/api.md)
- **Any QA Flow MCP write** → also [ai-rules/safeguard.md](ai-rules/safeguard.md)

The reference files in [ai-rules/reference/](ai-rules/reference/) (step-type guides, context wiring, flow structure, selection format) are read on demand exactly when the negotiation rule tells you to.

## The non-negotiable constraints (full detail is in the files above)

1. **No tool calls before negotiation.** The FIRST response to a flow/API request must be a single negotiation question — not `health_check`, not `list_*`, not `get_*`, not `validate_*`. "Gathering context first" is the forbidden pattern itself.
2. **One topic per message.** High-stakes decisions get their own question; related low-priority parameters (overrides+delays, exports+imports, all params of one query, polling cadence) are **grouped into a single message with preset options** per the rule. Never bundle unrelated concerns, assumptions, or a multi-step plan into one reply. Wait for each answer.
3. **Negotiate every step / every concern**, even when the API, test case, or a similar flow already exists. Existing artifacts are reference only. Each step type has a **full question set** in the rule (e.g. four messages for `api_call`: API, test case, overrides+delays, context wiring) — cover all of it, not just API/test-case.
4. **Get explicit confirmation before building.** Present the plan for confirmation only *after* every step/concern has been negotiated one question at a time — never as an upfront "here's my plan" that skips the per-step negotiation. Then confirm before calling `create_flow` / `update_flow` / `create_or_update_api`.
5. **After building, ask (as its own message) whether to run it** — never auto-run.

If you find yourself about to call a qa-flow MCP tool and you have not yet completed negotiation, stop and ask the next negotiation question instead.

## Session resilience (any assistant)

- Long negotiations can outlive the context window. If you can no longer see the rule file's full text or the answers gathered so far, **re-read the rule file** and restate the negotiated-answers ledger before asking the next question (safeguard Rule 7).
- Present choices per [ai-rules/reference/selection-format.md](ai-rules/reference/selection-format.md). If the client has a native single-question picker (e.g. AskUserQuestion), prefer it for every selection — small fixed sets as options directly; long lists via the hybrid pattern (full text table in the message + top 3–4 candidates as picker options) — still one question per message.
