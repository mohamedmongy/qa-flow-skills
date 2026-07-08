---
name: qa-flow-report-dashboard-negotiation
description: MUST be used whenever the user asks for a write or run action in the QA Flow report/analytics dashboard (📊 Reports tab) — tag/untag runs with a release (manage_releases), 🚀 execute a release's test groups, 🔄 run fresh & compare two releases (test_group_releases), 📤 export reports (export_test_group_results / load-test export), or clear the analytics cache (clear_analytics_cache) — including natural-language requests like "label these runs v1.2", "run release v2", or "export this report as PDF". Enforces negotiate-before-you-mutate-or-run — one topic per message (related low-priority settings grouped, e.g. release version + hidden-runs scope), no MCP tools before negotiation, explicit confirmation before applying. Does NOT apply to read-only viewing (get_analytics, cached release compare, viewing test-group / load-test results, listing releases or untagged jobs).
---

# QA Flow — Report Dashboard Negotiation

Acting on the QA Flow **reporting / analytics dashboard** (the 📊 **Reports** tab) is governed by strict negotiation rules. These rules are the source of truth — this skill exists only to make sure they are loaded and followed. **On any conflict between this summary and the rule files, the rule files win**; keep this summary in sync when the rules change.

## Do this first — read the authoritative rule

Before doing anything else, Read the matching file and follow it exactly:

- **Report dashboard write/run actions** → [ai-rules/negotiation/report-dashboard.md](ai-rules/negotiation/report-dashboard.md)
- **Any QA Flow MCP write** → also [ai-rules/safeguard.md](ai-rules/safeguard.md)

## The negotiated actions (the "action list")

| # | Dashboard feature | MCP tool(s) | Effect |
|---|---|---|---|
| 1 | Tag runs with a release | `manage_releases action='tag'` | Re-labels existing job runs (persistent) |
| 2 | Untag runs | `manage_releases action='untag'` | Removes the release stamp |
| 3 | 🚀 **Release Execution** | `test_group_releases action='by_release'` → `execute_test_group` per group | **Fires real test executions** |
| 4 | 🔄 **Run Fresh & Compare** | `test_group_releases action='execute_and_compare'` | **Re-runs both releases' groups**, then compares (long-running) |
| 5 | 📤 **Export Reports** | `export_test_group_results` / `get_load_test_results export=true` | Writes a report/result file |
| 6 | Clear the analytics cache | `clear_analytics_cache` | Discards cached analytics |

**Two distinct "release" systems live here — never conflate them:** a **job-level release tag** (`manage_releases`, written into a run's `payload`) powers the *analytics* comparison; a **test-group release** (`test_group_releases`, a `release` field on a test group) powers the dashboard's 🚀 Release Execution and 🔄 Release Comparison dropdowns.

Viewing any report, comparing releases **from cache**, listing releases, or viewing ⚡ **Load Test Results** is **read-only** and is **not** covered by this rule — do those directly.

## The non-negotiable constraints (full detail is in the files above)

1. **No tool calls before negotiation.** The FIRST response to a tag/untag/execute/run-fresh/export/clear request must be a single negotiation question (which action). Not `manage_releases`/`test_group_releases` (list/by_release), not `get_job_history`, not `get_analytics`, not `health_check`. "Gathering context first" is the forbidden pattern itself.
2. **One topic per message.** High-stakes decisions (action, runs/groups, environment) get their own question; related low-priority parameters are grouped (release version + include-hidden in one message; the two releases of a compare in one message). Never bundle unrelated concerns or a plan into one reply. Wait for each answer.
3. **🚀 Release Execution and 🔄 Run Fresh & Compare are real executions** — they run real tests against the active environment (run-fresh re-runs BOTH releases' groups). Negotiate the release set and the environment with the care of running a test group or load test; offer the cached compare first.
4. **Tagging is persistent and silently destructive** — it overwrites a run's stored `release_version` with no warning, and unknown `job_id`s are silently skipped. Always flag overwrites and reconcile the returned count against what you sent.
5. **Get explicit confirmation before applying.** Present the plan only *after* every concern is negotiated one question at a time. Then confirm before calling the write/run tool.
6. **After applying, report the real result** — tagged/untagged count (may be lower than requested), execution `job_id`s to monitor, or the exported file path — as its own message. Never auto-chain another action.

If you find yourself about to call a qa-flow report-dashboard write/run tool and you have not yet completed negotiation, stop and ask the next negotiation question instead.

## Session resilience (any assistant)

- Long negotiations can outlive the context window. If you can no longer see the rule file's full text or the answers gathered so far, **re-read the rule file** and restate the negotiated-answers ledger before asking the next question (safeguard Rule 7).
- Present choices per [ai-rules/reference/selection-format.md](ai-rules/reference/selection-format.md). If the client has a native single-question picker (e.g. AskUserQuestion), prefer it for every selection — small fixed sets as options directly; long lists via the hybrid pattern (full text table in the message + top 3–4 candidates as picker options) — still one question per message.
