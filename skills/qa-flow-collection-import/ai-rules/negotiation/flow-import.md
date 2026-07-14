# QA Flow — Flow Import (Steps Document) Rule

## Scope
These rules apply to **any AI assistant** whenever a user requests creating a QA Flow **flow** from a **fully-specified steps document** — a manual-test / test-steps file (`.txt`/`.md` or pasted text) whose steps carry explicit technical bindings: HTTP method + endpoint + payload, raw SQL + parameters, subflow references, `{{env.*}}` / `{{context.*}}` wiring, wait/poll cadences, and expected results. The recognized document format and its step idioms are defined in `reference/steps-doc-format.md` — read it before parsing.

This rule governs the **import workflow**. It does **not** replace `negotiation/flow.md`, `negotiation/query.md`, or `negotiation/api.md` — it **composes with them**: it substitutes their per-entity question cadence with a *parse → resolve → per-step confirmation loop → one summary + build confirmation* workflow, while every artifact it produces must still conform structurally — the flow to `flow.md`'s structure and step semantics (and the `reference/` files it points to), every saved query to `query.md`, every API definition to `api.md`. All writes obey `safeguard.md`.

**Eligibility gate.** The document must be *fully specified* per the checklist in `reference/steps-doc-format.md`: every step must classify into a QA Flow step type **and** carry the bindings that step type needs. A natural-language tester table (actions + expected results, no endpoints/SQL/wiring) is **not** eligible — negotiate it per `flow.md` instead. If only **some** steps lack bindings, the import may proceed with those steps flagged in the internal ledger; their missing details are asked in that step's turn of the loop (per `flow.md`'s question set for its step type), never silently guessed.

---

## Core Principle — Parse, Resolve, Walk the Steps, Build Once

**Never call `create_flow` — or any prerequisite write (`manage_environment_variables`, `save_query`, `create_or_update_api`) — before the per-step loop has covered every step and the user has explicitly confirmed the final build summary.**

The step ledger is the **agent's internal working state, never a deliverable**: do not display the full ledger (or a whole-flow plan) to the user. Its job is to drive the loop — the user sees exactly **one step's question at a time**: reuse what exists, or create what's missing according to its type. When every step is configured, ONE compact summary + ONE build confirmation authorize the whole chain. Never zero questions, never silent creation.

Always go through the steps below in order.

---

## Step 1 — Get the Source (Only If Missing)

If the request does not already point to a steps document (a file path, an attached file, or pasted text), the first response is — with **zero tool calls**:

> *"Point me at the steps document to import — a test-steps file or pasted steps with their endpoints/SQL and wiring."*

**Skip this entirely when the source is already provided** — never re-ask for what's in the request.

---

## Step 2 — Parse the Document into the Internal Ledger (Local Read — No MCP)

Read the file locally (allowed; not an MCP call) and classify **every** numbered step into a QA Flow step type using the idiom mapping in `reference/steps-doc-format.md`:

- HTTP verb + URL + payload + "Expect NNN" → `api_call`
- SQL + params → `query` (must exist as a **saved query**)
- "subflow X" → `flow` step (sub-flow invocation)
- "wait until / poll, timeout Ts every Is" → `wait_until`
- "Guard / Assert … else fail (halt)" → `conditional` (visual or `python_code`, failing branch → `raise_error`)
- "waits Ns before/after" → `waitBefore` / `waitAfter` on the adjacent step
- "same query as step N" → reuse of one saved query by a second step; "run twice" → two steps sharing one query
- "→ capture X" / "keep X as Y" → `context_export`

Extract in the same pass: the flow name (from the document header, else derive a `snake_case` proposal), the goal/description, the target environment / base URL, every `{{env.*}}` referenced, every `{{context.step.field}}` edge, and every static literal embedded in SQL, payloads, or conditions.

Record it all in the **internal ledger** — one row per document step: step name, type, target entity, wiring (params/imports → exports), and open gaps. In the same pass, build the **producer→consumer wiring graph** and validate it per *Context Wiring Integrity* below — every dangling edge becomes a ledger gap, resolved during the loop. The ledger drives the Step 4 loop and the Step 5 summary; it is **never shown to the user as a whole**.

Run the resolution pass (Step 3) **before** replying — the first response to an eligible document is the Step 4 question for the **first step**, never a bare summary, a full plan, or the raw ledger.

---

## Step 3 — Resolution Pass (the Only Permitted Pre-Confirmation MCP Calls)

This rule explicitly permits **one read-only resolution batch** after the document is parsed — the analog of `collection-import.md`'s single pre-negotiation name check. Exactly these, nothing else:

1. `list_flows` — name-availability check for the flow name (collision → new name per `flow.md` 2a, never overwrite).
2. `list_queries` — match each SQL step against existing saved queries.
3. `list_api_definitions` — match each HTTP step against existing API definitions.
4. `get_test_cases` — on each **matched** API, to check whether the document's named/implied test case exists.
5. `get_flow` — on each **referenced subflow**, to verify it exists and its declared `inputs` match what the document passes.
6. `get_environment` — existence check for every `{{env.*}}` the document references.

**No write tool, no `validate_*`, no run tool, and no other read tool runs before the final build confirmation.** Anything beyond this enumerated batch is the forbidden "gathering context first" pattern.

An existing entity counts as a **match** only when its content agrees with the document (same statement/params for a query; same method + endpoint for an API). Same name but different content is a **collision**, not a match.

Annotate every ledger row with the outcome: ✅ existing match / ❌ new / ⛔ blocked (missing subflow) / ⚠️ collision or near-match.

---

## Step 4 — Per-Step Confirmation Loop (ONE Step Per Message)

Walk the internal ledger in document order. For each step, ask **one question** — prefer the client's native structured picker (e.g. `AskUserQuestion`; see `reference/selection-format.md`) over plain-text menus, with the recommended option first:

- **Target exists (✅ match):** confirm reuse — e.g. `(1) Reuse <name> (Recommended)  (2) Create a new one under a new name  (3) Adjust`. Near-matches (same purpose, different content) are presented as a pick-one with the differences named; a same-name collision's create option proposes the suffixed/scoped new name.
- **Target missing, self-describing (❌ query / API / test case):** confirm creating it per its type — show the derived name and the bindings parsed from the document (statement + params / method + endpoint + payload + expected status), structurally conforming to `query.md` / `api.md`. The user's yes on the step authorizes adding it to the build list (built later, in Step 6 order — nothing is written during the loop).
- **Sub-flow step whose flow is missing (⛔):** a **blocking gap** — present the three *Entity Handling* options; the loop parks at this step until the user resolves it.
- **Everything that step still needs is asked inside its one message:** missing env-var **values** it references (sensitive ones marked), unresolved placeholders, defaulted poll cadences, static-literal placements (per `flow.md`'s *Static Values → Flow Inputs or Env Vars* — the step's literals as one table, never one question per literal), the step's wiring per *Context Wiring Integrity* below (resolved imports shown, exports shown with their consumers, dangling-edge repairs), and — for partially-specified documents — that step's `flow.md` questions.
- **Shared entities confirm once.** When a later step reuses an entity confirmed earlier ("same query as step N", "run twice"), don't re-ask the entity — either fold both steps into the first occurrence's question (naming both steps there) when the doc binds them identically, or ask only the later step's own wiring/exports.

One step per message, strictly in order; the user's answer updates the ledger; then the next step's question. Never batch several steps' decisions into one message (the shared-entity fold above is the only exception), never present the remaining ledger as a plan, and never call any write tool during the loop. The loop ends only when **every** ledger row is configured **and the wiring graph has no dangling edges**.

### Context Wiring Integrity

This section explicitly inherits `flow.md`'s *Smart Context Wiring* guarantees — above all: **never leave a dangling `{{context.*}}` reference.** Concretely:

1. **Build and validate the graph at parse time (Step 2).** Every consumed `{{context.<step>.<field>}}` must trace to a producing step whose `context_export` captures `<field>` (DB results as `result.<field>` per `reference/step-query.md`), to a flow input, or to `{{context.env.*}}`. An unmatched consumer is a **dangling edge** — a ledger gap, never something to silently build.
2. **Repair dangling edges at the *producing* step's turn.** When step N consumes a field no earlier step exports, the export is **retro-proposed at the producer's question** ("step 13 will also export `id` — step 17 needs it as `campaign_id`"), or — when no step can produce it — offered as a flow input / env var per *Static Values → Flow Inputs or Env Vars*. If the producer's turn has already passed when the gap surfaces, ask the repair as part of the consuming step's message, naming the producer it amends.
3. **Show the wiring on both sides of each loop question.** Each step's message displays its **resolved imports** ("`campaign_id` ← `{{context.db_whatsapp_campaign.id}}` from step 13") and its **exports with their consumers** ("exports `wa_message_id` → step 21"), so a wrong capture is caught at the producer — not ten steps later. Wiring the document already specifies is shown for confirmation, not re-asked open-ended.
4. **Renames propagate immediately.** Flow step names are the context prefixes: when a step is renamed during the loop (user choice, collision resolution), rewrite **every** downstream `{{context.<oldname>.*}}` reference ledger-wide before asking the next question. The Step 5 summary lists these rewires under loop decisions.
5. **Final gate before the summary:** re-walk the graph; if any edge is still dangling, the loop is not finished — resolve it before presenting the Step 5 summary. `validate_flow` (Step 6) is a backstop, not the first line of defense.

---

## Step 5 — Summary + Build Confirmation (ONE Message)

Only after the loop covers all steps, present one **compact summary** — decisions, not the raw ledger:

1. **Header** — flow name, one-sentence description, target environment / base URL / database.
2. **Reused** — the existing queries / APIs / subflows the flow will use (names only).
3. **To create** — missing env vars (with the values gathered during the loop; secrets masked), new saved queries (deduped — steps sharing SQL share one query), new APIs and additive test cases on existing APIs — counts and names.
4. **Loop decisions that changed the parse** — renames, collision resolutions, static-value promotions, adjusted cadences, resolved subflow blockers.
5. **Flow-level config** — shared session (on by default; proposed `session_config` headers), flow inputs, auth wiring (e.g. login subflow's token via `header_import`).
6. **The confirmation question** — e.g.: *"Build it? One yes authorizes, in order: N env vars → M saved queries (each smoke-tested) → K APIs → the flow `<name>` → `validate_flow`. Or tell me what to change."*

If the user amends anything, update the ledger and **re-confirm** before building. If the conversation is long enough that earlier answers scrolled out of view, restate the configured-steps checkpoint before building (`safeguard.md` Rule 7).

---

## Step 6 — Build (Only After Explicit Confirmation, in Dependency Order)

The summary's explicit "yes" authorizes the **whole chain** — no per-entity re-negotiation. Build strictly in this order, under `safeguard.md`:

1. **Env vars** — `manage_environment_variables` for every missing `{{env.*}}` (values gathered during the loop; the dashboard rejects artifacts referencing absent env vars).
2. **Saved queries** — `save_query` per `query.md`'s structure, then **smoke-test each immediately** (`test_query`). A failed smoke test **pauses the build**: report which query failed and why, mark the ledger checkpoint (✅ built / ⏸ pending), and wait for the user's decision (fix / skip the step / abort). Nothing downstream of the failure is built meanwhile.
3. **API definitions** — `create_or_update_api` per `api.md`: `{{env.BASE_URL}}` endpoints, one test case from the document's payload + expected status, auth via `header_import` (never hardcoded tokens). Adding a test case to an existing API is **additive** — never touch its other test cases.
4. **The flow** — **one** `create_flow` call with all steps, wiring, inputs, and session config as confirmed.
5. **Validate** — `validate_flow` immediately after creation; report the result.

---

## Step 7 — Report, Then Never Auto-Run

Summarize what was built (env vars, queries, APIs, the flow, validation result, anything skipped/renamed), then ask — as its own message, per `flow.md` Step 5:

```
The flow "<flow_name>" has been created successfully. Would you like to run it now?
```

Never run anything without that explicit yes.

---

## Entity Handling — New vs Existing

**Self-describing entities are synthesized from the document; non-self-describing ones block.**

- **Self-describing** (env vars, saved queries, API definitions, test cases): the document carries everything needed to create them — they are confirmed at their step's turn in the loop and built under the Step 5 confirmation. Only env-var **values** and embedded-literal **placements** need the user, asked inside the owning step's question.
- **Subflows are NOT self-describing.** The document names them (e.g. "subflow `login_v1`") without defining their internals — a missing subflow can never be guessed or synthesized. It is a **blocking gap**, presented at that step's turn with three options: **(a)** the user points at that subflow's own steps document → import it first (depth-first, this same rule); **(b)** build it via standard `flow.md` negotiation before resuming; **(c)** the user supplies the details to inline its steps into this flow. The parent import stays parked at that step until resolved.
- **Collisions never overwrite.** Same name + different content → create under a new suffixed/scoped name and rewire every ledger reference to it. Updating an existing artifact is a separate, explicit user request — never a fallback.
- **Idempotent re-import.** Same name + matching content → reuse as-is (offered as the recommended pick at that step). Re-importing an already-imported document creates nothing new.
- **Partial failure = checkpoint, never silent rollback.** If a build step fails, stop, report the ✅/⏸ ledger state, and wait. Already-created artifacts are **not** auto-deleted (deletes are safeguard-gated and usually still wanted); on the user's go-ahead, resume from the failed entity, not from scratch.

---

## What This Rule Is Not

- It is not a substitute for `negotiation/flow.md` — a flow described only as a goal or an informal step list (no bindings) is negotiated there, step by step.
- It does not apply to Postman/Swagger collections (use `negotiation/collection-import.md`) or to information-only requests ("summarize this steps doc", "what would this import create?" — parse locally and answer, no resolution pass, no negotiation).
- It does not permit silent creation — every step is confirmed in the loop, the build summary is confirmed explicitly, the build is dependency-ordered, and nothing is auto-run.
- It does not permit dumping the internal ledger or a whole-flow plan on the user — the ledger is agent state; the user gets one step per message, then one summary.

> **On any conflict between this file and `flow.md` / `query.md` / `api.md` / `safeguard.md`, those rules win for the concern they own** (flow structure and step semantics → `flow.md`; query structure → `query.md`; API structure → `api.md`; write/run safety → `safeguard.md`). This file owns only the steps-document import orchestration.
