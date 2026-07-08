# qa-flow-skills

Source of truth for the AI agent skills used by the
[QA Flow test-automation framework](https://gitlab.com/monty-mobile1/mobile-development/automation/qa-flow-test-automation).
They enforce QA Flow's negotiate-before-you-build and write-safeguard rules for
any coding agent driving the qa-flow MCP server.

**Generated content — do not edit here.** Edit `.claude/skills/` and `ai-rules/`
in the framework repo, then rebuild with `scripts/build_skills_repo.py`.

## Install

From your QA Flow project root (requires Node.js):

```bash
npx skills add mohamedmongy/qa-flow-skills
```

Install for specific agents only, non-interactively:

```bash
npx skills add mohamedmongy/qa-flow-skills -a claude-code -a cursor -y
```

Update / list / remove later:

```bash
npx skills update
npx skills list
npx skills remove
```

## Skills

| Skill | Purpose |
|---|---|
| `qa-flow-flow-negotiation` | MUST be used whenever the user asks to create or update a QA Flow flow (create_flow/update_flow) or API definition (create_or_update_api) via the qa-flow MCP server. |
| `qa-flow-load-test-negotiation` | MUST be used whenever the user asks to start or manage a QA Flow load test / stress test / performance test (start_load_test/manage_load_test_queue) via the qa-flow MCP server. |
| `qa-flow-query-negotiation` | MUST be used whenever the user asks to create, update, duplicate, or delete a QA Flow saved database query (save_query/update_query/duplicate_query/delete_query). |
| `qa-flow-report-dashboard-negotiation` | MUST be used whenever the user asks for a write or run action in the QA Flow report/analytics dashboard (📊 Reports tab). |
| `qa-flow-test-group-negotiation` | MUST be used whenever the user asks to create or update a QA Flow test group (create_test_group/update_test_group/modify_test_group_items) via the qa-flow MCP server. |
| `qa-flow-write-safeguard` | MUST be used whenever the user asks for any QA Flow MCP write, destructive, or execution action NOT covered by a negotiation skill. |

Each skill folder bundles the complete assistant-neutral rule set (`ai-rules/`),
so every skill is self-contained regardless of which subset you install.
The same rules are also served live by the qa-flow MCP server as
`qa-flow://rules/*` resources.
