---
name: sql-antipatterns-portable-sql

description: Cautions against restricting SQL to a hoped-for "portable" common subset across database brands — it both fails (vendors don't implement even standard features identically) and costs you each brand's genuinely useful proprietary features — and recommends an Adapter-pattern abstraction layer with brand-specific implementations instead.
---

# SQL Antipatterns — Chasing Portable SQL

This skill helps evaluate a specific engineering trade-off: trying to write
SQL that works unmodified across every database brand, versus writing
brand-specific SQL behind an abstraction layer.

---

## 1. Recognize the Situation

- A team restricts itself to what it believes is the common subset of SQL
  supported by every database brand it might ever use, in order to keep
  the codebase portable — sometimes for a database switch that isn't even
  planned, just hedged against.
- This is a real, well-intentioned engineering instinct, not a schema
  design mistake — but it rests on two assumptions worth checking before
  committing to it.

---

## 2. Why It Doesn't Deliver What It Promises

- **It gives up real, useful features for a hypothetical.** Every vendor
  adds proprietary extensions beyond the standard — window functions,
  `GROUP_CONCAT()`/array aggregates, `ANY_VALUE()`, engine-specific index
  types, and more. Restricting to only universally-supported syntax means
  never using any of them, even when one would make a query dramatically
  simpler or faster, in exchange for a portability guarantee that isn't
  actually being delivered (see next point).
- **It doesn't actually work.** The SQL standard defines "levels" of
  compliance, and vendors implement different subsets — and even for
  features every vendor claims to support, implementations diverge in
  behavior, not just syntax. Even core, unglamorous things like standard
  data types aren't implemented identically across brands. Code written
  against what looks like "plain standard SQL" can still behave
  differently when pointed at a different database, defeating the purpose
  of avoiding brand-specific code in the first place.
- The standard itself is written in English prose, not a formal
  specification — vendors can (and do) interpret ambiguous wording
  differently in good faith, which is part of why "supports the standard"
  doesn't guarantee identical behavior.

---

## 3. The Fix: Isolate Brand Differences Behind an Abstraction

Rather than writing one lowest-common-denominator query and hoping it
behaves identically everywhere, isolate the parts of the codebase that
touch the database behind an interface, and provide a brand-specific
implementation per supported database (the Adapter design pattern). Each
implementation is free to use that brand's actual best features —
window functions, native array/JSON handling, brand-specific
optimizations — instead of being constrained to whatever the weakest
supported brand can do.

This costs more upfront (one implementation per supported brand instead
of one shared query) but delivers what "portable SQL" was actually trying
to promise: code that behaves correctly and takes advantage of each
database's real capabilities, instead of code that merely looks portable
while quietly depending on unverified cross-brand behavior.

---

## 4. Review Checklist

- Is SQL being restricted to a "common subset" for a database migration
  that isn't actually planned or realistically likely?
- Has "portable" syntax actually been tested against every target brand,
  or is portability assumed because the syntax looks standard?
- Would a proprietary feature (window functions, brand-specific
  aggregates, native JSON/array support) make a query meaningfully
  simpler or faster than the self-imposed portable subset allows?
- If multiple database brands genuinely need to be supported, is there an
  abstraction boundary where brand-specific implementations can diverge
  safely, instead of one shared query trying to serve every brand at once?
