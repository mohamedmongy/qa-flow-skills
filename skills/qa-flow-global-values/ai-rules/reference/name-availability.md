# QA Flow — Name Availability (shared)

This is the single authoritative copy of the name-availability rule referenced by every rule that checks whether an artifact name is free before creating something: `negotiation/api.md`, `negotiation/flow.md`, `negotiation/query.md`, `negotiation/test-group.md`, `negotiation/collection-import.md`, `negotiation/flow-import.md`, `negotiation/spec-plan.md`, and `operations/plan-execute.md`.

---

## The rule in one line

**A `list_*` result proves a name is TAKEN. It never proves a name is FREE.**

Before creating anything, a name that *looks* free in a listing must be confirmed with that artifact's **targeted lookup**.

---

## Why — absence in a listing is not evidence of absence

Tool results are capped (20,000 characters). When a listing exceeds the cap it is trimmed, and items past the cut are **not in the text you are reading**. An artifact that exists but was trimmed away looks exactly like one that does not exist — so "I don't see it in the list" silently becomes "the name is free", and the result is a duplicate artifact plus a missed reuse the user wanted.

This is not hypothetical. On a mature project the API and query listings routinely overflow: this repo's own dashboard has hidden `login_with_otp`, `login_with_otp_v2`, `csrf_demo` and `csrf_v2` behind the cap — precisely the auth APIs a spec with an `Authorization` header needs to reuse.

A trimmed listing says so: it carries `"truncated": true` with `showing`, `total`, `omitted`, and a `note`. **Read those fields before drawing a conclusion from a listing** — but do not rely on their absence either, since a listing that fits today can overflow tomorrow as the project grows.

---

## What to do instead

### 1. Use the compact name list for name checks

Every name-check listing accepts `names_only=True`, which returns just the names. It is roughly a tenth of the size, so it stays complete far longer:

```
list_api_definitions(names_only=True)   list_queries(names_only=True)
list_flows(names_only=True)             list_test_groups(names_only=True)
```

Use the full listing when you need each item's detail (method, endpoint, counts) to *match* an artifact against a cURL or statement; use `names_only` when the question is only "is this name taken?".

### 2. Confirm every name you intend to create with its targeted lookup

| Artifact | Targeted lookup | Exists | Free |
|---|---|---|---|
| API definition | `get_api_definition(name)` | returns the definition | fails with 404 |
| Saved query | `get_query(name)` | returns the query | fails with 404 |
| Flow | `get_flow(name)` | returns the definition | fails with 404 |
| Test group | `get_test_group(name)` | returns the group | fails with 404 |
| Prerequisite template | `get_prerequisite_template(name)` | returns the template | fails with 404 |

A 404 is the **only** reliable evidence that a name is free.

### 3. This confirmation is part of the permitted pre-negotiation check

The negotiation rules allow exactly one read-only name-availability check before negotiation begins. **The targeted lookup for the candidate name is part of that same check** — not an extra "gathering context first" pass, which remains forbidden. Concretely, the permitted sequence before the name question is:

```
list_*(names_only=True)            → is the candidate visible as taken?
    ↓ candidate not visible
get_<artifact>(candidate)          → 404 = genuinely free · 200 = collision
```

Checking a handful of candidate names this way (one per artifact the request will create) is still one check. Walking the whole listing calling `get_*` on every existing item is not — that is context-gathering, and it is not permitted.

### 4. Report honestly

- Never tell the user a name is available on the strength of a truncated listing.
- Never conclude "no API matches this endpoint" from a truncated listing alone — endpoint matching needs the targeted lookups too.
- When a listing came back truncated and it matters to the answer, say so, and say what you did about it.

---

## Collisions

When the targeted lookup finds the name taken, the owning rule's collision handling applies unchanged — a collision **always** resolves to a new name, never to an overwrite or a silent update. See `negotiation/api.md` (Step 1c), `negotiation/flow.md` (Step 2a), `negotiation/query.md`, and `negotiation/test-group.md` for each artifact's alternative-name conventions.
