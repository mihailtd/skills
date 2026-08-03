---
name: python-fastapi-partial-updates

description: Guides implementing PATCH-style partial updates in FastAPI so a client that omits a field never accidentally clears it — using Pydantic's exclude_unset (backed by model_fields_set) to distinguish "field omitted from the request" from "field explicitly set to null," since a naive full-object overwrite treats every unset field as if the client wants it cleared, silently destroying data written by any other client, an older API version, or a consumer that only knows about a subset of fields.
---

# FastAPI Partial Update Guidelines

You are an expert FastAPI developer specializing in resource-update
endpoints. When asked to implement an endpoint that updates an existing
resource, adhere to the following rules — a naive full-object overwrite is
one of the most common, and most silent, ways a FastAPI service loses data.

## 1. The Data-Loss Trap: Full-Object Overwrite

A resource-update endpoint that copies every field from the request model
onto the stored record treats every field the client *didn't* send the same
as a field the client explicitly wants cleared — because an all-optional
Pydantic model fills unset fields with their default (usually `None`):

```python
# DANGEROUS: any field the client omitted becomes None, wiping existing data
class PersonUpdate(BaseModel):
    name: str | None = None
    occupation: str | None = None

@router.put("/people/{person_id}")
async def update_person(person_id: str, body: PersonUpdate):
    person = await get_person(person_id)
    person.name = body.name              # None if the client didn't send it
    person.occupation = body.occupation  # None if the client didn't send it
    await save_person(person)
```

This is especially costly across API versions and heterogeneous clients: an
older client, a partial-schema mobile client, or any consumer that simply
doesn't know a field exists sends a request with that field absent — and a
full-object overwrite erases whatever value was already there, even though
the client never intended to touch it. This is the same "complete only from
the caller's point of view" trap covered generally in
**architecture-network-api-versioning**.

## 2. The Fix: `exclude_unset=True` to Get Only What Was Actually Sent

Pydantic tracks which fields were present in the input via
`model_fields_set` — `model_dump(exclude_unset=True)` uses that to return
only the fields the client actually included, omitting anything that fell
back to a default:

```python
class PersonUpdate(BaseModel):
    name: str | None = None
    occupation: str | None = None

@router.patch("/people/{person_id}")
async def update_person(person_id: str, body: PersonUpdate):
    person = await get_person(person_id)
    update_fields = body.model_dump(exclude_unset=True)  # only sent fields
    for field, value in update_fields.items():
        setattr(person, field, value)
    await save_person(person)
```

If the client sends `{"name": "Eric"}`, `update_fields` is `{"name": "Eric"}`
— `occupation` is absent from the dict entirely and is never touched,
regardless of what it currently holds.

## 3. Omitted vs. Explicitly Null Are Different Requests — Keep Them Different

`exclude_unset=True` is what makes it possible to distinguish two requests
that would otherwise look identical on an all-Optional model:

- `{"name": "Eric"}` — the client is silent about `occupation`; leave it
  untouched.
- `{"name": "Eric", "occupation": null}` — the client explicitly wants
  `occupation` cleared; that field *is* present in `model_fields_set`, with
  a value of `None`, so it *is* included by `exclude_unset=True` and should
  be applied.

Both cases pass through the same `model_dump(exclude_unset=True)` call
correctly — the distinction is already encoded in which keys are present in
the returned dict, not something that needs separate handling. Don't
"simplify" this by filtering out `None` values after the fact — doing so
silently reintroduces the exact bug this pattern exists to prevent, by
throwing away a client's explicit clear request.

## 4. Applying the Same Pattern to the Persistence Layer

Pass the same only-what-was-sent dict through to whichever persistence layer
is in use — never reconstruct a "complete" object from the request model
and write that wholesale:

```python
# SQLAlchemy: targeted column update, not a full-row replace
from sqlalchemy import update

await session.execute(
    update(PersonModel)
    .where(PersonModel.id == person_id)
    .values(**body.model_dump(exclude_unset=True))
)

# Beanie: .set() with only the provided fields
await person_doc.set(body.model_dump(exclude_unset=True))
```

This keeps the "only touch what was actually sent" guarantee intact all the
way to storage, rather than reintroducing a full-overwrite risk at the
database layer even after getting it right at the API layer.

## 5. Route Design: PATCH for Partial, PUT (if Offered) for True Full Replace

Use `PATCH` for this pattern — it's the HTTP method whose semantics are
specifically "apply these changes," matching what `exclude_unset=True`
implements. If a genuine full-replace endpoint (`PUT`) is also needed,
implement it as an explicit, separate, and clearly-documented behavior —
don't let a `PUT` endpoint quietly behave like a full overwrite by accident
while assuming callers understand that tradeoff. When only one update
endpoint is offered, default to `PATCH` semantics for any resource where an
unrelated field's data has real value — which is most of them.

A partial update built this way is also naturally idempotent — sending the
same `PATCH` request twice produces the same end state both times, since
it only ever sets the fields it names to the values it names. See
**architecture-idempotency-and-at-least-once-delivery** for why that
property matters for retry-safety.

## Related guidance

- **architecture-network-api-versioning** (package `architecture`) — the
  broader versioning-strategy reasoning (client-controlled vs.
  server-controlled) this pattern serves, and why it matters more under
  server-controlled versioning specifically.
- **architecture-idempotency-and-at-least-once-delivery** (package
  `architecture`) — why a well-formed partial update is naturally
  idempotent and safe to retry.
- **python-fastapi-routing-validation** — general request-body validation
  patterns this skill's `PersonUpdate`-style models build on.
