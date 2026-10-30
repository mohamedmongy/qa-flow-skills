# QA Flow — Selection Format Rule (shared)

This is the single authoritative copy of the selection-format rule referenced by all negotiation rules (flow, API, test group, query, load test, report dashboard). When presenting choices to the user during negotiation, match the format to the size of the choice set.

## Small / fixed choice sets

Action menus, yes-type picks, a handful of options → an **inline numbered menu on a single line** so the user can reply with just a number:

```
(1) flow input   (2) env var   (3) keep it hardcoded
```

## Long lists of existing items

Flows, APIs, test cases, test groups, available items, runs, releases, databases, … → a **clear, readable table/list**, one option per line, each numbered, with a short identifying detail per row (type, count, status, date) so the rows are easy to tell apart.

1. **Cap at 20:** if there are **fewer than 20** options, show them all. If there are **20 or more**, show only the **first 20**, then add a line telling the user that more exist and to just name the one they want (or ask to see more).
2. **Multi-select indicator:** when the user may pick more than one item, prefix every row with a checkbox `- [ ]` so it is visually clear that several can be selected.

Example — long list, multi-select:

```
There are ~30 groups total — if the one you want isn't here, just tell me the name.
- [ ] (1) nadim_mcp_merged_group — 2 flows
- [ ] (2) nadim_demo_group — 1 flow (smoke)
- [ ] (3) mcp_test_group_v1 — 1 flow
- [ ] (4) gcrsf_all — 1 test suite
```

## Grouped low-priority parameters (preset menus)

**Do not ask related, low-stakes parameters one-by-one — that is overhead, not negotiation.** When several parameters together configure a single concern, collect them in **one message**, preferably as a preset menu:

- A **load profile**: users + spawn rate + duration
- A **polling cadence**: timeout + interval (+ condition)
- **Group settings**: stop-on-failure + release + tags
- A query's / flow's **parameter set**: all parameters in one table-style question
- **Static-value placements**: all pending literals in one table (each picks flow input / env var / hardcode)

Format: named presets as an inline menu, each showing **all** its values, plus a **Custom** option:

```
How much load? (1) Light — 10 users, 2/s ramp, 1m   (2) Default — 100 users, 10/s ramp, 5m
(3) Heavy — 500 users, 20/s ramp, 10m   (4) Custom — tell me users / spawn rate / duration
```

- Picking a preset counts as **explicit confirmation of every value in it** — this is how defaults get consented to, never silently applied.
- If the user picks **Custom**, they supply the values in one reply; ask only for whatever is missing.
- If the user answers only part of a grouped question, ask only for the missing parts — don't repeat the whole menu.

**What must still be its own single question** (never grouped with anything):
the target/action being performed, the environment/host real traffic will hit, any destructive-intent confirmation, and the final plan confirmation.

## Native structured-question UI (any assistant)

If the client/assistant provides a native single-question option picker (e.g. Claude Code's `AskUserQuestion` tool, an IDE quick-pick), **prefer it over plain-text menus for every negotiation selection** — it enforces one-question-per-message, gives one-click answers, and includes a free-text "Other" escape. Map each format above onto the picker like this:

- **Small / fixed choice sets** → picker options directly (up to the picker's option cap, typically 4; "Other" covers the rest).
- **Long lists of existing items** → the **hybrid pattern**: print the full list as a text table in the message body (the cap-at-20 rule and `- [ ]` multi-select indicator still apply), then attach the picker carrying the **3–4 most likely candidates** from that list — the user one-clicks a candidate, or picks "Other" and types any name from the table. Never silently shrink a long list to only the picker's options; the full table must stay visible so the picker never hides choices.
- **Grouped preset menus** → one preset per picker option, each label carrying **all** its values (as in the load-profile example above), plus a Custom option. Picking a preset counts as explicit confirmation of every value in it, same as the text form.
- **Multi-select** → if the picker supports selecting several options natively, use that for pick-several questions; otherwise fall back to the `- [ ]` text table.

Constraints that always apply:

- **One question per prompt, always.** Never use a multi-question form to bundle several negotiation topics into one turn — the one-question-per-message rule still applies.
- **Seed the options from context.** When the negotiation context suggests likely answers — a similar existing item found during context-gathering, the user's earlier answers in this negotiation, or established project patterns — make those the options instead of generic placeholders (e.g. offer *"same endpoint as `csrf`"*, the reference API's tag/priority combo, or common status codes for the scenario). Put the recommended option first and mark it *(Recommended)*. The picker's built-in "Other" keeps free-text available, so seeding options never traps the user.
- **Genuinely open questions stay free-text** — names, descriptions, scenario wording, pasted cURLs, and the values behind a "Custom" pick have no meaningful options to seed; ask them as plain questions.
- If no such UI exists (plain chat), use the text formats above — they are the portable default.
