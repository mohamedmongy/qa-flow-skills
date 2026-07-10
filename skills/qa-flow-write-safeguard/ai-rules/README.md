# ai-rules/ — Assistant-Neutral QA Flow Rules (Source of Truth)

This folder is the **single source of truth** for how ANY AI assistant must behave when driving the QA Flow MCP server — negotiating before building flows, APIs, test groups, and queries, before firing load tests, and before mutating the report dashboard. The rules are assistant-neutral: Kiro, Claude Code, GitHub Copilot, Cursor, Codex, and any future assistant all follow **these same files**.

## Layout

| Path | What it is |
|---|---|
| [negotiation/](negotiation/) | The strict negotiate-before-you-build/run rules, one per domain (flow, api, test-group, query, load-test, report-dashboard) |
| [safeguard.md](safeguard.md) | Cross-cutting MCP write rules (Rules 1–7): tool-only edits, root-cause first, destructive-write confirms, context-loss recovery |
| [reference/](reference/) | On-demand material the negotiation rules point to: step-type guides, context wiring, flow structure, selection/grouping format |
| [manifest.json](manifest.json) | Machine-readable index: every rule's id, path, kind, and the MCP tools it gates |

## How each assistant consumes this folder

| Consumer | Mechanism |
|---|---|
| Any agent with the QA Flow skills installed | `npx skills add mohamedmongy/qa-flow-skills` — installs the intent-triggered skills (built from `.claude/skills/` + this folder by `scripts/build_skills_repo.py`) to Claude Code, Kiro CLI, Cursor, Codex, Copilot, and ~70 other agents; each skill bundles its own copy of these rules |
| Claude Code / AGENTS.md readers without the skills | `CLAUDE.md` / `AGENTS.md` fallback — the always-loaded mandatory-rules table pointing here |
| Any MCP client | the qa-flow MCP server serves this folder live as `qa-flow://rules/manifest` + `qa-flow://rules/{rule-id}` |

**A future assistant needs no code changes:** install the skills with `npx skills add`, point it at `AGENTS.md`, or read `manifest.json` / the MCP rules resources programmatically.

## Editing rules

1. **Edit the rule here** — never in a skill or fallback doc, and never duplicate rule content there.
2. Skills and fallback docs need touching only when **triggers or paths** change (new rule, renamed file, new gated tool).
3. In the framework dev repo, sync to the CLI templates per `.kiro/steering/cli-package-sync-rule.md`, republish the skills repo with `scripts/build_skills_repo.py`, and run the drift check (`cli_package/tests/test_ai_rules_sync.py`) plus the behavior checklist (`project_context/negotiation-rules-testing-checklist.md`).
4. In a generated user project, this folder is **framework-owned**: `qa-flow upgrade` overwrites it — don't hand-edit.

## The one-paragraph version of the rules

Before any QA Flow MCP **write** (create/update/delete/run): the FIRST response to the user is a **single negotiation question** (no tool calls first — not even read-only ones; single exception: flow/API creation runs one `list_flows`/`list_api_definitions` name-availability check before the opening name question, suggesting alternatives on collision); **one topic per message**, with related low-priority settings grouped into a single preset-menu question; **every concern negotiated** even when similar artifacts exist; a **full plan confirmed explicitly** before building; and after building, **ask before running** — never auto-run. Destructive tools additionally require confirming the exact target and its dependents (safeguard Rule 6).
