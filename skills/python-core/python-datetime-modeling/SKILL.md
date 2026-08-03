---
name: python-datetime-modeling

description: Instructs the agent on modeling date/time data correctly in Python — distinguishing an instant (an absolute moment, aware datetime), a civil date (a calendar date with no time attached, e.g. a birthdate), a civil datetime bound to a place (a naive local datetime plus a separate IANA zone name, for future-scheduled events), a duration (timedelta, always fixed elapsed time), and a period (calendar-relative arithmetic via dateutil.relativedelta) — the aware-vs-naive datetime trap, IANA zone names vs UTC offsets vs abbreviations, and why calendar arithmetic isn't associative or reversible.
---

# Python Date and Time Modeling Guidelines

You are an expert Python developer specializing in correct date/time handling. When asked to model, store, compare, or compute with date/time data, adhere to the following rules — most date/time bugs come from conflating concepts that are genuinely different, not from a specific API being used wrong.

## 1. Pick the Right Concept, Not Just the Right Type

Four distinct concepts get conflated constantly. Identify which one a piece of data actually is before choosing a representation:

- **Instant** — an absolute moment everyone agrees on regardless of location (an event timestamp, `created_at`, a log entry). Represent as a **timezone-aware** `datetime.datetime`.
- **Civil date** — a calendar date with no time-of-day and no time zone at all (a birthdate, an anniversary, a "due by" date). Represent as `datetime.date`, never `datetime.datetime`. Attaching a time and/or a zone to a pure calendar date invites the exact bug you're trying to avoid: converting it through a time zone can shift it onto the "wrong" calendar day near midnight.
- **Civil datetime bound to a place** — a wall-clock date+time tied to a specific location whose rules can still change before it happens (a meeting scheduled six months out for 2 p.m. in New York). Represent as a **naive** `datetime` plus a **separate** `ZoneInfo` (IANA zone name) field, and resolve them into an absolute instant only when you actually need one — not once, baked in, at write time. The zone's DST rules between now and the event date aren't fixed, so precomputing the UTC instant today can go stale.
- **Duration vs. period** — a duration is a fixed amount of elapsed time, always the same regardless of calendar context (`datetime.timedelta`); a period is calendar-relative ("1 month," "3 years") and its actual elapsed time depends on when it's applied. Python's stdlib has no first-class period type — use `dateutil.relativedelta.relativedelta` for calendar-relative arithmetic, and never use `timedelta(days=30)` to mean "1 month."

Getting this wrong at the model layer can't be fixed by careful code later — see **database-postgresql-temporal-modeling** / **database-mongodb-temporal-modeling** for making the same distinction at the schema level, which is where it actually needs to start.

## 2. Always Use Timezone-Aware Datetimes for Instants

`datetime.now()` and `datetime.utcnow()` (deprecated as of 3.12, and subtly wrong even before then — it returns a *naive* value with no `tzinfo`, so nothing marks it as UTC) both return **naive** datetimes — Python's version of the "naive timestamp" trap already guarded against at the database layer (see **database-postgresql-timestamp-timezone-management**).

```python
from datetime import datetime, timezone
from zoneinfo import ZoneInfo

# correct: aware, unambiguous
now_utc = datetime.now(timezone.utc)
now_ny = datetime.now(ZoneInfo("America/New_York"))

# wrong: naive — no tzinfo, ambiguous the moment it leaves this process
now_naive = datetime.now()
```

Python raises `TypeError` when comparing an aware datetime to a naive one — treat this as a safety net, not an obstacle. Fix the naive value at its source (make it aware) rather than stripping `tzinfo` from the aware one to make the comparison "work."

## 3. Use IANA Zone Names — Never Fixed Offsets or Abbreviations

`ZoneInfo("America/Los_Angeles")`, not a fixed `timezone(timedelta(hours=-8))` and never an abbreviation string like `"PST"`:

- A fixed offset doesn't track a location's DST transitions — it's frozen at whatever offset applied when you wrote it.
- Abbreviations are actively ambiguous — `BST` means British Summer Time today, but meant British Standard Time between 1968 and 1971; `PST`/`PDT` aren't time zone identifiers at all, just descriptions of the offset San Francisco's actual time zone (`America/Los_Angeles`) happens to observe part of the year.

```python
from zoneinfo import ZoneInfo

tz = ZoneInfo("America/Los_Angeles")  # correct — an actual IANA zone
```

The stdlib `zoneinfo` module (3.9+) reads IANA tzdata from the OS. On platforms that don't ship it (notably some Windows installs), install the `tzdata` package explicitly (`uv add tzdata`) — without it, `ZoneInfo(...)` raises at runtime instead of silently falling back to something wrong, which is the right failure mode, but worth knowing about before it happens in production.

## 4. Calendar Arithmetic Is Not Associative or Reversible

Adding a period to a date has no single universally-correct answer once you hit a month/year boundary — different reasonable interpretations exist, and they don't compose the way normal arithmetic does:

```python
from datetime import date
from dateutil.relativedelta import relativedelta

d = date(2021, 1, 31)

# (d + 1 month) + 2 months
step1 = (d + relativedelta(months=1)) + relativedelta(months=2)  # 2021-04-28

# d + (1 month + 2 months)
step2 = d + (relativedelta(months=1) + relativedelta(months=2))  # 2021-04-30
```

Both are "correct" by the library's own rules — they're just not equal to each other, because clamping February 31st down to February 28th at the intermediate step loses information the second calculation never had to discard. The same non-reversibility applies to subtraction: adding a month to January 31st and then subtracting a month back does not reliably return January 31st.

**Concrete consequence:** any business rule involving calendar arithmetic near a month/year boundary — eligibility dates, subscription renewals, age-based cutoffs — needs its exact rule spelled out explicitly (which of the reasonable interpretations applies) and tested at the edge case, never assumed obvious from the requirement's plain-English wording. "Customers can return items within 3 months" sounds unambiguous until an order lands on October 31st.

## 5. Correctness Starts at the Schema, Not Just in Application Code

Application code that correctly distinguishes instant/civil-date/civil-datetime-plus-zone still inherits a bug if it's reading from a schema that didn't make the same distinction — a birthdate column stored as a UTC timestamp, for instance, can already have shifted to the wrong calendar day before your Python code ever sees it. Get the column type right at the schema level first: see **database-postgresql-temporal-modeling** and **database-mongodb-temporal-modeling**.

## 6. Apply the Same Concept Consistently Across Memory, Network, and Storage

The same piece of date/time data typically crosses three boundaries — in-memory objects while code runs, text in network requests/responses, and a storage representation — and it's easy to let the concept quietly drift between them (a value that's genuinely a civil date in memory but gets treated as an instant the moment it's serialized). Keep the concept identical at every boundary: a user-selected date that arrives as `"2024-12-20"` should parse straight into a `date`, stay a `date` throughout the code, and land in a `DATE` column — not get promoted to a `datetime`/timestamp at some point along the way just because that happened to be convenient.

Two disciplines make this easy to keep straight:

- **Parse incoming data into your preferred in-memory type as early as possible, and serialize outgoing data into the destination format as late as possible** — this minimizes how much code has to deal with an inconsistent representation, since the "boundary" conversion is a single, narrow seam instead of something scattered through business logic.
- **Centralize the conversion code itself** rather than repeating the same parse/format logic at every call site — if the representation ever needs to change (a new API version, a schema migration), there should be exactly one place to update, not a search-and-replace across the codebase.

When the target representation is structurally poorer than your in-memory concept (a datastore that only has a single timestamp type, no separate date-only type — see **database-postgresql-temporal-modeling** and **database-mongodb-temporal-modeling** for how PostgreSQL and MongoDB specifically handle this), that's a deliberate representation choice to make explicitly and document, not something to solve ad hoc at each call site.

## 7. Choosing a Library

The stdlib combination of `datetime` (with explicit `tzinfo`) and `zoneinfo.ZoneInfo` is the default, and is sufficient for the large majority of applications — it's immutable, IANA-based, and already ships with Python 3.9+. Reach for `dateutil.relativedelta` specifically for calendar-relative (period) arithmetic, since the stdlib has no first-class period type (section 1). Only consider a third-party library beyond that (`pendulum`, `arrow`) when a specific, concrete gap justifies the extra dependency — nicer ergonomics alone rarely outweighs adding another library to a concern this easy to get subtly wrong; if you do evaluate one, check specifically whether it supports IANA zones (not just fixed offsets), provides immutable values, and offers the same concept distinctions this skill relies on before adopting it.

## Related guidance

- **database-postgresql-temporal-modeling** / **database-mongodb-temporal-modeling** — the same instant/civil-date/civil-datetime-plus-zone distinction applied to schema design, where it should actually start.
- **database-postgresql-timestamp-timezone-management** — `TIMESTAMPTZ` mechanics and the naive-timestamp trap at the PostgreSQL level.
- **python-datetime-requirements-clarity** — resolving what a date/time requirement actually means before applying this skill's concept model to it.
- **python-datetime-testability** — injecting the current time and time zone explicitly instead of relying on system defaults, once the concept model here is in place.
- **python-testing-mocking** — controlling "now" in tests that exercise date/time logic.
