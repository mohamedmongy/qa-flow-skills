---
name: qa-flow-test-group-negotiation
description: MUST be used whenever the user asks to create or update a QA Flow test group (create_test_group/update_test_group/modify_test_group_items) via the qa-flow MCP server — including natural-language requests like "bundle these flows", "make a regression/smoke suite or group", "add/remove/reorder items in a group", or "tag a group with a release". Enforces the project's negotiate-before-you-build rules — one topic per message (items+order in one question; stop-on-failure/release/tags grouped as one settings question), never call MCP tools before negotiation, get explicit confirmation before building. Does NOT apply to information-only requests (list/show groups or results) or to deleting/copying groups (see qa-flow-write-safeguard).
---

# QA Flow — Test Group Negotiation

Creating or updating QA Flow **test groups** in this repo is governed by strict negotiation rules. These rules are the source of truth — this skill exists only to make sure they are loaded and followed. **On any conflict between this summary and the rule files, the rule files win**; keep this summary in sync when the rules change.

## Do this first — read the authoritative rule

Before doing anything else, Read the matching file and follow it exactly:

- **Test group** (`create_test_group` / `update_test_group` / `modify_test_group_items`) → [ai-rules/negotiation/test-group.md](ai-rules/negotiation/test-group.md)
- **Any QA Flow MCP write** → also [ai-rules/safeguard.md](ai-rules/safeguard.md)

A test group bundles items that must **already exist** — `flow`, `test_suite`, and `query`. If a needed item doesn't exist yet, create it first under its own negotiation rule (flow / API / query), then add it to the group.

## The non-negotiable constraints (full detail is in the files above)

1. **No tool calls before negotiation.** The FIRST response to a test-group request must be a single negotiation question (name + description) — not `list_test_groups`, not `get_test_group_available_items`, not `health_check`, not any `get_*`. "Gathering context first" is the forbidden pattern itself.
2. **One topic per message.** Name gets its own question; **items + execution order are one grouped question**, and **stop-on-failure + release + tags are one grouped settings question** with defaults shown. Never bundle unrelated concerns or a multi-step plan into one reply. Wait for each answer.
3. **Negotiate every concern** — items, order, execution config, release, tags — even when a similar group already exists. Existing groups are reference only.
4. **Get explicit confirmation before building.** Present the plan for confirmation only *after* every concern has been negotiated one question at a time — never as an upfront "here's my plan" that skips negotiation. Then confirm before calling `create_test_group` / `update_test_group` / `modify_test_group_items`.
5. **After building, ask (as its own message) whether to run it** — never auto-run. If the user says yes, negotiate the run options (environment, background/foreground) per the rule before executing.

If you find yourself about to call a qa-flow test-group write tool and you have not yet completed negotiation, stop and ask the next negotiation question instead.

## Session resilience (any assistant)

- Long negotiations can outlive the context window. If you can no longer see the rule file's full text or the answers gathered so far, **re-read the rule file** and restate the negotiated-answers ledger before asking the next question (safeguard Rule 7).
- Present choices per [ai-rules/reference/selection-format.md](ai-rules/reference/selection-format.md). If the client has a native single-question picker (e.g. AskUserQuestion), prefer it for every selection — small fixed sets as options directly; long lists via the hybrid pattern (full text table in the message + top 3–4 candidates as picker options) — still one question per message.
