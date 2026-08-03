---
name: database-mongodb-temporal-modeling

description: Guides date/time field modeling in MongoDB given that BSON's Date type is always an absolute instant (UTC milliseconds since epoch) with no native "civil date" or "local datetime without a zone" type — storing a birthdate or a future local-wall-clock event as a Date risks the same timezone-shift bugs as misusing TIMESTAMPTZ in PostgreSQL, and the fix is a plain ISO-8601 string for anything that isn't genuinely an instant, plus a separate IANA zone-name field for future local events.
---

# MongoDB Temporal Modeling — Working Around a Single Native Date Type

PostgreSQL gives you `TIMESTAMPTZ`, `TIMESTAMP` (no zone), and `DATE` as
distinct types, so the instant/civil-date/civil-datetime distinction (see
**database-postgresql-temporal-modeling**) can be enforced by the schema
itself. MongoDB doesn't have that luxury: BSON's `Date` type is **always** an
absolute instant — internally, milliseconds since the Unix epoch, UTC, with
no concept of "no time zone attached." There is no BSON equivalent of `DATE`
or a naive `TIMESTAMP`. Modeling the other temporal concepts correctly means
deliberately *not* reaching for `Date` when the field isn't actually an
instant.

## 1. Use `Date` Only for Genuine Instants

`created_at`, `updated_at`, `payment_processed_at` — anything that answers
"when did this happen, as an absolute moment" — should be a BSON `Date`:

```javascript
{
  _id: ObjectId("..."),
  order_id: "abc123",
  created_at: new Date()  // instant — correct use of Date
}
```

Note BSON `Date` only has millisecond precision (not micro/nanosecond) —
rarely an issue, but worth knowing if you ever need to break ties between
events that happened within the same millisecond.

## 2. Don't Store a Civil Date as `Date`

A birthdate, an anniversary, a deadline expressed as "by this calendar day"
has no time-of-day and no time zone as part of what it means — but storing
it as a `Date` forces one in anyway (conventionally UTC midnight), and that
choice is exactly what creates the bug: reading that value back and
displaying it in a different assumed time zone can shift it onto the wrong
calendar day, the identical failure mode covered for PostgreSQL's
`TIMESTAMPTZ` misuse in **database-postgresql-temporal-modeling**.

Store a pure civil date as a plain ISO-8601 date string instead — there's
then no `Date` type present to tempt any driver, ORM, or downstream
consumer into running a time zone conversion over data that was never an
instant in the first place:

```javascript
{
  _id: ObjectId("..."),
  customer_id: "u_42",
  birthdate: "2004-02-29"  // string, not Date — no zone to shift through
}
```

## 3. For a Future Local Event, Store Wall-Clock Time and Zone Separately

The same "civil datetime bound to a place" case from
**database-postgresql-temporal-modeling** applies identically here — a
meeting scheduled for 2 p.m. in New York, six months out, shouldn't be
collapsed into a single UTC instant at write time, because that bakes in
today's assumption about New York's DST rules on that future date. Store the
wall-clock components and the IANA zone name as separate fields, and resolve
them into an instant only when actually needed:

```javascript
{
  _id: ObjectId("..."),
  title: "Quarterly review",
  local_datetime: "2025-06-15T14:00:00",  // string — no "Z", no offset, not a Date
  iana_zone: "America/New_York"
}
```

Using a plain string (not `Date`) for `local_datetime` is deliberate, for the
same reason as the civil-date case: a `Date` value here would misleadingly
imply "this is an instant," inviting exactly the kind of premature,
stale-DST-assumption resolution this representation exists to avoid.

## 4. Resolve to an Instant Only at Read Time, in Application Code

Combine `local_datetime` and `iana_zone` into an actual instant only when the
consuming code needs one — in Python, parse the string and attach
`ZoneInfo(iana_zone)` (see **python-datetime-modeling**) rather than storing
a precomputed instant anywhere. This guarantees the resolution always uses
whichever DST rules are current *as of when you ask*, not whichever were
current when the document was written.

For the same instant/civil-date/civil-datetime-plus-zone framework applied to
a database with distinct types to enforce it, see
**database-postgresql-temporal-modeling** (package `database-postgresql`);
for general BSON schema design this skill's field-type choices fit within,
see **database-mongodb-schema-modeling-patterns**.
