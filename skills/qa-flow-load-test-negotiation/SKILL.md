---
name: qa-flow-load-test-negotiation
description: MUST be used whenever the user asks to start or manage a QA Flow load test / stress test / performance test (start_load_test/manage_load_test_queue) via the qa-flow MCP server — including natural-language requests like "hammer this endpoint", "run 100 users against X", "stress the checkout flow", or "promote/cancel a queued load test". Enforces the project's negotiate-before-you-run rules — one topic per message (related low-priority settings like users/spawn-rate/duration grouped into one preset question), never call MCP tools before negotiation, get explicit confirmation before firing real load. Does NOT apply to viewing status or results (get_load_test_status / get_load_test_results without export).
---

# QA Flow — Load Test Negotiation

Starting or managing QA Flow **load tests** in this repo is governed by strict negotiation rules. These rules are the source of truth — this skill exists only to make sure they are loaded and followed. **On any conflict between this summary and the rule files, the rule files win**; keep this summary in sync when the rules change.

## Do this first — read the authoritative rule

Before doing anything else, Read the matching file and follow it exactly:

- **Load test** (`start_load_test` / `manage_load_test_queue`) → [ai-rules/negotiation/load-test.md](ai-rules/negotiation/load-test.md)
- **Any QA Flow MCP write** → also [ai-rules/safeguard.md](ai-rules/safeguard.md)

A load test runs against a target that must **already exist** — a `test_group`, a `flow`, or an `api`/test suite. If the target doesn't exist yet, create it first under its own negotiation rule (flow / API / test group), then load test it.

## The non-negotiable constraints (full detail is in the files above)

1. **No tool calls before negotiation.** The FIRST response to a load-test request must be a single negotiation question (which target?) — not `list_test_groups`, not `list_flows`, not `get_load_test_status`, not `health_check`, not any `get_*`. "Gathering context first" is the forbidden pattern itself.
2. **One topic per message.** Negotiate target → **load profile** (users + spawn rate + duration as ONE grouped preset-menu question — never three separate questions) → environment + queue-or-immediate (one message). Wait for each answer.
3. **Never apply the defaults silently.** Default load is **100 concurrent users** fired at a real `BASE_URL`; the load-profile presets show all values, and picking one is the explicit consent — this is real, destructive-in-effect traffic.
4. **Targets must already exist** — never invent a flow/api/suite/group name. Only `flow` and `test_suite` items execute; `query` items in a group are silently skipped. `queue=false` (immediate) works only for test-group targets.
5. **Get explicit confirmation before starting.** Present the plan only *after* every concern has been negotiated, then confirm before calling `start_load_test` / `manage_load_test_queue`.
6. **After starting, ask (as its own message) whether to monitor it** — never auto-poll into results or chain another run. Results don't exist until the job finishes.

If you find yourself about to call a qa-flow load-test write tool and you have not yet completed negotiation, stop and ask the next negotiation question instead.

## Session resilience (any assistant)

- Long negotiations can outlive the context window. If you can no longer see the rule file's full text or the answers gathered so far, **re-read the rule file** and restate the negotiated-answers ledger before asking the next question (safeguard Rule 7).
- Present choices per [ai-rules/reference/selection-format.md](ai-rules/reference/selection-format.md). If the client has a native single-question picker (e.g. AskUserQuestion), prefer it for every selection — small fixed sets as options directly; long lists via the hybrid pattern (full text table in the message + top 3–4 candidates as picker options) — still one question per message.
