# Steps Document Format — One-Shot Flow Import

> Read when `negotiation/flow-import.md` sends you here: this file defines what a **fully-specified steps document** looks like, how its idioms map to QA Flow step types, and the eligibility checklist that decides whether a document qualifies for one-shot import. It is also the **authoring contract**: testers who follow this format get clean one-shot imports.

---

## Document Skeleton

```
<Title> — <what the test covers> (<flow_name>)
Goal: <one-sentence end-to-end description>

Notes for the reader
{{env.X}} = environment value. {{context.step.field}} = a value captured from an earlier step.
Base URL used in <env>: <https://...>
DB = <engine> <database> (schemas: ...)

A. <Phase title>
1. <Step title> (<step kind hint>)
   <binding: SQL / HTTP call / subflow name / poll spec>
   params: <name>={{env.X}}, ... → capture <field>.
2. ...

B. <Phase title>
...

Overall pass criteria
<what "all steps passed" means>
```

- **Title line** — carries the flow name, ideally as a trailing `(snake_case_name)`; otherwise the import derives and proposes one.
- **Goal line** — becomes the flow description.
- **Notes block** — declares the notation, the base URL / target environment, and the database. `{{env.X}}` and `{{context.step.field}}` are the QA Flow template syntax used verbatim.
- **Lettered phase sections** (`A.`, `B.`, …) — grouping/naming hints only; steps are numbered continuously across them.
- **Numbered steps** — one per flow step (with the idiom exceptions below). The step title becomes the step name (`snake_case`d) unless the doc names one explicitly.

---

## Step Idioms → QA Flow Step Types

| Document idiom | Step type | Required bindings |
|---|---|---|
| HTTP verb + URL (+ payload) + "Expect NNN" | `api_call` | method, full URL (env-templated), payload/params, expected status |
| SQL statement + `params:` | `query` | the statement, every parameter's source (`{{env.*}}` / `{{context.*}}` / literal) |
| "(subflow `X`)" / "subflow `X`" | `flow` | the subflow name + the inputs passed to it |
| "wait for … (`wait_until`, timeout Ts / every Is)" | `wait_until` | what to poll (SQL or API), timeout, interval, pass condition |
| "Guard: …" / "Assert … else fail/halt" / "(conditional)" | `conditional` | condition (field + operator + value, or described python logic), on-true / on-false outcomes |
| "waits Ns after/before" | — | `waitAfter` / `waitBefore` on the adjacent step |
| standalone "wait Ns" step | `wait` | the duration |

**Cross-cutting idioms**

- `→ capture X` / `keep X as Y` → `context_export` on that step (DB results export as `result.<field>`; `Y` names the context key when given).
- `same query as step N` → the second step **reuses** step N's saved query (one query, two steps).
- `run twice` / "(second check)" → two flow steps sharing one binding.
- "Pass only if … AND/OR …" → the conditional's logic operator; "the test fails/stops" → `raise_error` (or `stop_flow` when phrased as halt-without-failure).
- All-caps/emphasized **before**/**after** snapshots (e.g. "balance before") → the export key names (`balance_before`, `consumed_after`, …).

---

## Eligibility Checklist — Is the Document Fully Specified?

A document qualifies for one-shot import when **every step** satisfies all of:

1. It classifies unambiguously into one step type via the table above.
2. It carries that type's required bindings (endpoint+payload / SQL+params / subflow+inputs / poll spec+condition / condition+outcomes).
3. Every value it consumes is traceable — a `{{env.*}}`, a `{{context.step.field}}` produced by an earlier step's capture, or an explicit literal.
4. Its expected result is stated (expected status, pass condition, or assertion).

A document of **actions + expected results only** (a tester table: "Log in with the YAPAS user → Login is successful") fails 2 and is **not eligible** — negotiate it per `negotiation/flow.md`. A document where only some steps fail the checklist is **partially eligible**: import proceeds with those steps flagged in the gap report and negotiated individually before confirmation.

**Wiring note on item 3:** a consumed `{{context.<step>.<field>}}` whose `<field>` no earlier step captures is a **dangling edge**, not an eligibility failure — the import repairs it during the loop by retro-proposing the export at the producing step's turn (see `negotiation/flow-import.md`, *Context Wiring Integrity*), never by building the dangling reference. Authors: make every capture explicit (`→ capture X` / `keep X as Y`) and reference steps by the exact name the capture line implies.

---

## Minimal Example

```
Wallet debit check (wallet_debit_check)
Goal: Log in, snapshot the balance, charge the wallet, and verify the exact deduction.

Notes: Base URL (QA): https://api.qa.example.net — DB = PostgreSQL billing_db.

A. Setup
1. Login (subflow login_v1) → capture accessToken used by all API calls.
2. Balance BEFORE (DB)
   SELECT current_balance FROM billing.client_account WHERE client_id = %s
   param client_id={{env.CLIENT_ID}} → keep current_balance as balance_before.

B. Charge & verify
3. Charge wallet — POST {{env.BASE_URL}}/api/v1/wallet/charge
   { "Amount": 5, "ClientId": "{{env.CLIENT_ID}}" }
   Expect 200; capture transactionId.
4. Wait for transaction row (wait_until, timeout 30s / every 2s)
   SELECT id, status FROM billing.transaction WHERE external_id = %s
   param external_id={{context.charge_wallet.transactionId}} → pass when a row appears.
5. Balance AFTER (DB — same query as step 2) → keep current_balance as balance_after.
6. Assert deduction (conditional python) — balance_after == balance_before - 5, else fail.
```

This imports as: 1 `flow` step + 2 `query` steps sharing one saved query + 1 `api_call` + 1 `wait_until` + 1 `conditional`, with `balance_before`/`balance_after` exports, one new API (`charge_wallet`), two new saved queries, and the `Amount: 5` literal run through the *Static Values → Flow Inputs or Env Vars* promotion.
