---
name: database-postgresql-temporal-modeling

description: Guides column-type selection for date/time data based on what concept a column actually represents — an instant (TIMESTAMPTZ), a pure civil date with no time zone (DATE), or a civil datetime bound to a specific place whose future offset isn't fixed yet (a naive TIMESTAMP plus a separate IANA zone-name column, resolved at read time) — correcting the common but incomplete rule of "always use TIMESTAMPTZ," and covering the same non-associativity of calendar arithmetic in SQL that application code has to deal with.
---

# PostgreSQL Temporal Modeling — Picking the Right Type Per Concept

`TIMESTAMPTZ` is the correct default for most timestamp columns — see
**database-postgresql-timestamp-timezone-management** for why naive
`TIMESTAMP` is a trap. But "always use `TIMESTAMPTZ`" is a rule that stops one
step too early: not every date/time column represents the same *concept*, and
picking the type from the concept, not from a blanket default, is what
actually prevents the subtler bugs. This skill covers that decision.

## 1. Three Temporal Concepts, Three Column Choices

- **Instant** — an absolute moment everyone agrees on, regardless of where
  they are: `created_at`, `payment_processed_at`, a log entry. Use
  `TIMESTAMPTZ`. This is the case the existing `TIMESTAMPTZ`-by-default rule
  correctly covers.
- **Civil date** — a calendar date with no time-of-day and no time zone at
  all: a birthdate, an anniversary, a "valid until" date. Use `DATE`, never
  `TIMESTAMPTZ`. A `DATE` column has no time zone to convert through, so it
  can't shift to the wrong calendar day the way a timestamp-typed birthdate
  can when displayed in a session time zone different from where it was
  entered.
- **Civil datetime bound to a place** — a wall-clock date+time tied to a
  specific location, for something that hasn't happened yet: a meeting
  scheduled six months out for 2 p.m. in New York, a recurring local
  appointment. Use a naive `TIMESTAMP` (no time zone) **plus a separate
  column storing the IANA zone name** (`'America/New_York'`, as `TEXT`), and
  resolve the pair to an absolute instant only when you actually need one —
  at read time, not once at write time.

## 2. Why the Third Case Needs Two Columns, Not One `TIMESTAMPTZ`

`TIMESTAMPTZ` stores (and is only capable of storing) an absolute instant —
internally, a UTC value with no memory of which zone or offset produced it.
For a *future* local event, that's the wrong thing to bake in: the UTC offset
for `'2 p.m. in New York, six months from now'` depends on New York's DST
rules *at that future date*, not today's. Storing it as `TIMESTAMPTZ` computes
and freezes today's offset assumption immediately, so if the zone's rules
change between now and the event (a real, if infrequent, occurrence — DST
rules are set by governments and do change), the stored instant silently
stops corresponding to "2 p.m. in New York" and nobody's data changed to
tell you so.

```sql
-- wrong for a future local event: bakes in today's DST assumption
CREATE TABLE appointments (
    id SERIAL PRIMARY KEY,
    scheduled_at TIMESTAMPTZ NOT NULL
);

-- correct: naive wall-clock time + the place it's anchored to,
-- resolved to an instant only when actually needed
CREATE TABLE appointments (
    id SERIAL PRIMARY KEY,
    local_datetime TIMESTAMP NOT NULL,
    iana_zone TEXT NOT NULL  -- e.g. 'America/New_York'
);
```

Resolve the pair to an instant in application code (Python: combine with
`ZoneInfo(iana_zone)` — see **python-datetime-modeling**) or in SQL with `AT
TIME ZONE` at query time, so the resolution always uses whatever the zone's
rules are *as of the moment you ask*, not as of the moment you wrote the row.

## 3. The Civil-Date Mistake Is Subtler and Easy to Miss in Review

Storing a birthdate as `TIMESTAMPTZ` looks harmless — it "has a date in it."
The bug shows up specifically at the boundary: a birthdate of `2004-02-29`
stored as `TIMESTAMPTZ` at UTC midnight can display as `2004-02-28` in a
session time zone west of UTC, because midnight UTC on the 29th is still the
evening of the 28th in, say, `America/Los_Angeles`. A `DATE` column has no
such failure mode — there's no time zone to convert through, so the calendar
date it stores is the calendar date it returns, always.

Treat any column holding a pure calendar concept (birthdate, anniversary,
a deadline expressed as "by this day," a fiscal-year boundary) as a `DATE`
by default, and require a specific justification before widening it to a
timestamp type.

## 4. Calendar Arithmetic in SQL Has the Same Non-Associativity as Application Code

`date + interval '1 month'` follows defined rules, but — exactly like the
application-layer `relativedelta` arithmetic in **python-datetime-modeling**
— those rules produce results that don't compose the way plain numeric
arithmetic does. Adding a month past a shorter month clamps to that month's
last valid day, and the result depends on the order operations are applied
in, not just the total. Test calendar arithmetic explicitly at month/year
boundaries (the last day of February, the last day of any 30-day month)
rather than assuming a single "obviously correct" result — a query that
looks right for `2024-01-15 + interval '1 month'` is not proof it's right for
`2024-01-31 + interval '1 month'`.

For the same instant/civil-date/civil-datetime-plus-zone distinction applied
in application code, see **python-datetime-modeling** (package
`python-core`); for the same framework applied to a database with only one
native date type, see **database-mongodb-temporal-modeling** (package
`database-mongodb`).
