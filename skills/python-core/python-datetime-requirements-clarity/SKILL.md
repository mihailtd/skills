---
name: python-datetime-requirements-clarity

description: Instructs the agent to treat a vague date/time requirement ("customers can return items within 3 months") as underspecified until specific questions are answered — which instant a period is relative to, which location's time zone governs it (never the server's or database's physical location), the exact granularity and calendar-arithmetic rollover rule, whether boundaries are inclusive, and whether a grace period applies — and to retain the raw source instant even after deriving other representations, since the answers can change later.
---

# Clarifying Date/Time Requirements Before Writing Code

You are an expert at turning ambiguous date/time requirements into precise, testable specifications. When a requirement mentions a date, time, deadline, or duration, treat it as underspecified by default and resolve the following before writing logic — most date/time bugs trace back to a question nobody asked, not a coding mistake.

## 1. Identify the Source Concept Before Deciding How to Store It

- If the requirement describes **"something happened"** (an order was placed, an item shipped, a payment was accepted), the canonical value is an **instant** — capture it as a timezone-aware value the moment it occurs. Most frameworks and databases can timestamp this automatically; the open question is *which* event's timestamp is the relevant one; see section 2.
- If the requirement describes **a value a user provided** (a birthdate, a preferred delivery day, an appointment time they typed in), that's **civil time**, not an instant — retain what the user actually gave you (the calendar date/time as entered, plus whatever zone context came with it) rather than immediately converting it to a UTC instant. Converting early throws away information you may need later, particularly for anything dated in the future — see **python-datetime-modeling** section 1 for why a future civil datetime shouldn't be collapsed into a precomputed instant.

## 2. For Any Period ("within N days/months/years"), Resolve Five Specific Ambiguities

A requirement like "customers can return items within 3 months" sounds complete but leaves at least five questions unanswered. Don't guess — either get an explicit answer or state the assumption explicitly in the code, docstring, and tests:

1. **Relative to which instant?** A single vague trigger ("shipping") often hides several distinct real events — payment accepted, order confirmed, stock allocated, shipped, received. Get the specific one, and capture it as its own field even if it isn't the one the current rule uses — requirements about *which* instant matters tend to change, and you want the raw instant preserved so the rule can be recomputed later, not just its current derived output.
2. **Governed by which time zone?** Never anchor date/time behavior on the physical location of a server or database — a product should not behave differently because a VM happens to be running in a particular AWS region. Anchor on a location that's actually meaningful to the business rule (the delivery address, the user's declared time zone, the store's timezone) and get an explicit answer for which one.
3. **What granularity?** Date-only, or date-and-time? "3 months from a 10 a.m. shipment" expiring at exactly 10 a.m. three months later reads as arbitrary precision to a customer — many business rules genuinely only care about the calendar date, not the time-of-day, and should say so explicitly.
4. **What's the exact calendar-arithmetic rule at edge cases?** Adding "1 month" to the 30th or 31st of a month has no single universally-correct answer once the target month is shorter (see **python-datetime-modeling** section 4) — state explicitly whether the result rolls over to the next month's start, clamps to the target month's last day, or something else, and write a test for the specific boundary date (the last day of a 30- or 31-day month, and February) rather than only the common case.
5. **Is the boundary inclusive, and is there a grace period?** "Within 3 months" needs an explicit answer for whether the final day itself still counts, and whether a check performed slightly after the nominal deadline (because a user was mid-form, or a request was in flight) should still be honored. A user-facing grace period (e.g., "valid if checked within 5 minutes of when it was still valid") is a legitimate, common answer — but it has to be a stated decision, not an accident of implementation timing.

## 3. Write the Resolved Requirement as a Concrete, Testable Sentence

Once the five ambiguities are resolved, restate the requirement so every clause maps directly to a test case — this is the same document that becomes the docstring and the acceptance tests:

> The option to return an item is based on the date the item shipped, in the time zone of the delivery address. The last valid return date is computed by adding 3 months to the shipping date. If that calculation would land beyond the end of the target month, it rolls over to the 1st of the following month instead (e.g. shipping on November 30th makes the last valid date the following March 1st, not the last day of February). The return option is available so long as the current date at the delivery location is not later than the last valid return date.

Every sentence here answers one of the five questions from section 2 — the source date (shipping), the governing zone (delivery address), the granularity (date, not datetime), the rollover rule (roll to next month's start, with a worked example), and the boundary (inclusive, "not later than"). A requirement that can't be restated this precisely isn't ready to implement yet — go back and ask.

## 4. Capture Raw Source Data Even If the Current Logic Only Needs a Derived Value

Store every instant you can identify as relevant (order confirmed, stock allocated, shipped, delivered), even if today's rule is only defined in terms of one of them. Requirements about which instant governs a calculation change more often than the underlying events themselves — keeping the raw instants means a future rule change is a recompute, not a data migration to recover information that was never kept.

## Related guidance

- **python-datetime-modeling** — the concept model (instant/civil-date/civil-datetime/duration/period) these questions are grounded in, and why calendar arithmetic doesn't compose the way plain arithmetic does.
- **python-datetime-testability** — once the requirement is unambiguous, writing it as testable code with an injected clock and explicit time zone.
- **database-postgresql-temporal-modeling** / **database-mongodb-temporal-modeling** — capturing the resolved concept (instant vs. civil date vs. civil datetime-plus-zone) correctly at the schema level once you know which one you're storing.
