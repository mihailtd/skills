---
name: sql-antipatterns-rounding-errors

description: Detects the Rounding Errors antipattern — using FLOAT/REAL/DOUBLE PRECISION for money or other exact fractional values — which silently rounds decimal values that can't be represented exactly in IEEE 754 binary, breaking equality comparisons and accumulating error in sums/products, and guides toward NUMERIC/DECIMAL instead.
---

# SQL Antipatterns — Rounding Errors

This skill helps developers recognize when `FLOAT`/`REAL`/`DOUBLE PRECISION`
is being used for values that need to be exact — money above all — and
guides them to `NUMERIC`/`DECIMAL` instead.

---

## 1. Recognize the Antipattern

- **The tell:** a column meant to hold money or another value that must
  compare and sum exactly is declared `FLOAT`, `REAL`, or `DOUBLE
  PRECISION`:

  ```sql
  ALTER TABLE Accounts ADD COLUMN hourly_rate FLOAT;
  ```

- **Root cause:** `FLOAT` mirrors the name of the `float`/`double` types in
  most programming languages, so it's the reflexive choice for "a number
  with a decimal point" — but SQL's `FLOAT` carries the same IEEE 754
  binary encoding and the same inexactness those language types have.
- Not every fractional value needs this skill's fix — see §3 for when
  `FLOAT` is genuinely the right call.

---

## 2. Why It Breaks Down

- **Some exact decimal values have no exact binary representation.**
  IEEE 754 stores numbers in base 2. A value like `59.95` — completely
  exact and finite in decimal — requires infinite precision in binary, so
  it gets rounded to the nearest representable value, e.g.
  `59.950000762939`. This isn't a bug in a particular database; it's
  inherent to the format.
- **Equality comparisons silently fail.** Insert `59.95` into a `FLOAT`
  column and the stored value is already slightly off:

  ```sql
  SELECT * FROM Accounts WHERE hourly_rate = 59.95;
  -- returns nothing, even though you stored exactly 59.95
  ```

  The usual workaround — comparing `ABS(a - b) < threshold` — trades one
  problem for another: the "right" threshold varies per value and per
  calculation, and picking one that's too tight silently reintroduces the
  original failure.
- **Error accumulates across aggregates.** Each individual rounding error
  is small, but `SUM()` over many `FLOAT` values accumulates every one of
  them. Repeated multiplication (e.g. compounding interest) is worse:
  a per-step factor that's off by a fraction of a percent compounds --
  multiplying by 0.999 instead of 1.0 across a thousand iterations lands
  around 0.3677 instead of 1.0, not a rounding-error's worth off.
- **The failure is invisible until someone checks by hand.** Nothing
  errors — the report just doesn't match a manual calculation, and the
  discrepancy looks like "off by a few dollars," which is exactly the kind
  of mismatch that erodes trust in a system without an obvious root cause.

---

## 3. Legitimate Uses

- `FLOAT`/`DOUBLE PRECISION` are the right choice when you genuinely need
  IEEE 754's much wider dynamic range (very large and very small
  magnitudes together) — scientific and measurement data is the classic
  case (e.g. temperature readings).
- They're appropriate specifically when the *use* of the data tolerates
  inexactness: aggregate calculations (`MIN()`, `MAX()`, `AVG()`) and
  range queries (`BETWEEN`, inequality comparisons) degrade gracefully
  with small rounding error. The risk is concentrated in equality
  comparisons and anything that must reconcile against an independently
  computed exact value (like money against a hand calculation) — if
  nothing in the workload does that, `FLOAT` may be fine.
- Some databases blur the naming: Oracle's `FLOAT` is actually an exact
  scaled numeric, while `BINARY_FLOAT` is the inexact IEEE 754 type.
  Check what a given database brand's `FLOAT` actually means before
  assuming it matches the IEEE 754 behavior described here.

---

## 4. The Fix: Use NUMERIC/DECIMAL

Store fixed-precision fractional values — money above all — as `NUMERIC`
or `DECIMAL` (synonyms; `DEC` is also a synonym for `DECIMAL`):

```sql
ALTER TABLE Bugs ADD COLUMN hours NUMERIC(9,2);
ALTER TABLE Accounts ADD COLUMN hourly_rate NUMERIC(9,2);
```

- The first argument is **precision**: the total number of decimal digits
  the column can hold. The second is **scale**: how many of those digits
  sit right of the decimal point. `NUMERIC(9,2)` holds values up to
  `1234567.89`, not `12345678.91` (too much scale) or `1234567890` (too
  much precision).
- Precision/scale apply uniformly to every row in the column — you can't
  mix scale 2 on some rows and scale 4 on others, same as a `VARCHAR(20)`
  column can't allow longer strings on some rows.
- With `NUMERIC`, equality behaves the way you'd expect, because the value
  is stored exactly rather than rounded to the nearest representable
  binary fraction:

  ```sql
  SELECT hourly_rate FROM Accounts WHERE hourly_rate = 59.95;
  -- returns 59.95, reliably
  ```

- This doesn't grant infinite precision — a value like one-third still
  can't be stored exactly in `NUMERIC` either. The difference is that its
  rounding behavior matches decimal intuition (the same rounding you'd do
  by hand), instead of an unrelated set of values being silently perturbed
  because of a base-2 encoding.

---

## 5. Review Checklist

- Is any column storing money, or any value compared for exact equality,
  typed `FLOAT`/`REAL`/`DOUBLE PRECISION`?
- Are there `ABS(a - b) < threshold` comparisons standing in for equality
  — a sign the underlying column is the wrong type, not that the query
  needs a fuzzy-match workaround?
- Do report totals (`SUM()`, compounding calculations) ever come up
  "close but not exact" versus an independently computed value? That's a
  strong signal of accumulated floating-point rounding.
- For the specific database brand in use, does `FLOAT` actually mean IEEE
  754 binary floating point, or (as in Oracle) an exact scaled numeric
  under a confusing name? Verify rather than assume.
- If the workload only ever does range queries and aggregates — never
  equality comparisons or exact reconciliation — `FLOAT` may be a
  deliberate, correct choice; don't convert it reflexively.

---

## Related guidance

PostgreSQL-specific remedy:

- database-postgresql-avoid-legacy-data-types
