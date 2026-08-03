---
name: sql-antipatterns-storing-prices

description: Clarifies when storing a seemingly-redundant value (like an order's price) alongside a reference to its current source (like a product's price) is correct, not a normalization violation — because it captures a point-in-time fact in an association table, not a duplicate of an ever-changing one.
---

# SQL Antipatterns — Storing Prices (Point-in-Time Facts)

This skill helps developers tell apart a genuine normalization violation
(the same current fact stored in two places, able to disagree) from a
correct design (a historical fact captured at the moment an association
was made, which is expected to diverge from the current value over time).

---

## 1. Recognize the Situation

- A table records an association between two entities (an order between a
  customer and a product, a lineup between a player and a game, a cast
  list between an actor and a film) — i.e. it's an intersection/association
  table for a many-to-many relationship.
- One column on that association table appears to duplicate a value that
  "lives" in one of the referenced tables:

  ```sql
  CREATE TABLE Orders (
      order_id       SERIAL PRIMARY KEY,
      order_date     DATE NOT NULL,
      customer_id    INT NOT NULL,
      merchandise_id INT NOT NULL,
      quantity       INT NOT NULL,
      price          NUMERIC(9,2) NOT NULL  -- looks redundant with Merchandise.unit_price?
  );
  ```

- The instinct to flag this as a normalization violation ("shouldn't price
  live only in `Merchandise`, referenced by `merchandise_id`?") is
  reasonable to raise, but wrong here.

---

## 2. Why It's Not a Violation

- Normalization's rule is that each *fact* should be stored once. The
  question is whether `Orders.price` and `Merchandise.unit_price` are
  actually the same fact — they aren't.
- `Merchandise.unit_price` answers "what does this cost right now."
  `Orders.price` answers "what did this specific customer actually pay on
  this specific date" — after whatever discount, sale, or membership
  pricing applied at that moment. Prices change over time; sales and
  discounts mean even the price at that moment could differ from the
  catalog price. These are two different facts that happen to share a
  data type and a plausible-sounding column name.
- If you *derived* the order's price at query time from the current
  `Merchandise.unit_price` instead of storing it, historical orders would
  silently reprice themselves every time the catalog price changed — an
  actual data integrity bug, not the thing normalization is protecting
  you from.
- The same shape recurs anywhere an association needs to freeze a fact as
  of the moment it was formed: a sports team's game-day lineup (who was on
  the roster *that day*, not today), a film's cast credit (an actor's
  name as credited *then*, even if they're credited differently now).

---

## 3. The Rule of Thumb

- Ask: "if the referenced entity's current value changes tomorrow, should
  every past association silently change with it, or should it stay
  fixed at what was true when the association was recorded?"
  - If it should stay fixed → store it directly on the association/
    intersection table. This is correct design, not denormalization to
    apologize for.
  - If it should always reflect the current value → don't store it
    redundantly; look it up through the reference instead.
- This only applies to association-table columns that capture a fact
  *about that specific association at that specific time*. It's not a
  general license to duplicate current, mutable facts across tables —
  that's still a real normalization violation with the usual risk of the
  copies disagreeing.

---

## 4. Review Checklist

- Does the column in question represent "the value as of this
  association" or "the current value of the referenced entity"? Only the
  former belongs on the association table.
- If the referenced table's value changes, is there a business reason past
  records must NOT change retroactively (contracts, financial records,
  historical reporting, audit trails)? That's a strong signal the value
  belongs on the association table.
- Is this actually a plain duplicate of a *current, mutable* fact with no
  point-in-time meaning — i.e. genuinely redundant, not historical? Then
  treat it as a normalization problem, not an exception to it.

---

## Related guidance

PostgreSQL-specific remedy:

- database-postgresql-avoid-legacy-data-types
