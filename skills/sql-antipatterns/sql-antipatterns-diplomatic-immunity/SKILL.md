---
name: sql-antipatterns-diplomatic-immunity

description: Detects the Diplomatic Immunity antipattern — treating database code (schema, triggers, stored procedures, seed data) as exempt from the software engineering practices applied to application code — no documentation, no version control, no automated tests — and guides toward documenting the schema, checking database artifacts into the same version control as application code (with migration tooling), and writing automated tests against database structure and behavior.
---

# SQL Antipatterns — Diplomatic Immunity

This skill helps developers recognize when database code is being treated
as exempt from the documentation, version control, and testing discipline
applied everywhere else in a project, and guides them to close that gap
before technical debt in the database becomes the reason a project can't
be maintained by anyone but the one person who remembers it.

---

## 1. Recognize the Antipattern

- **The tell:** application code gets code review, tests, documentation,
  and version control as a matter of course — but the schema, triggers,
  stored procedures, and seed data live only in the live database, known
  only to whoever last touched them, with no written record of why they
  exist or how to reproduce them.
- **Listen for these phrases**:
  - "We're adopting the new engineering process — a lightweight version of
    it." — often a euphemism for skipping the parts that felt like too
    much friction, without a deliberate accounting of what's being traded
    away.
  - "The DBA team doesn't need training on our version control system,
    since they don't use it anyway." — a self-fulfilling exclusion that
    guarantees database artifacts stay outside the same discipline as
    everything else.
  - "How do we track which tables/columns hold PII/SPI for compliance?" —
    a direct symptom of missing schema documentation; without it, the only
    safe fallback is treating *all* data as sensitive, which is more
    costly than documenting it would have been.
  - "Is there a tool to diff two schemas and generate a reconciliation
    script?" — a sign schema changes aren't tracked through a controlled
    process, so environments have silently drifted apart.
  - "My code is self-documenting." — rarely true even for application
    code, and especially not for a schema: code can't tell you about
    features that don't exist yet, business rules a constraint encodes, or
    decisions that were deliberately *not* made.

---

## 2. Why It Breaks Down

- **Knowledge concentrated in one person is a single point of failure for
  the whole project**, not just an inconvenience — the chapter's framing
  story (a sole developer's death leaving an undocumented, unversioned,
  untested application behind) is the extreme case, but the everyday
  version is just as real: anyone leaving the team, changing roles, or
  simply forgetting details over time has the same effect at smaller
  scale.
- **Undocumented schema forces conservative, expensive defaults.**
  Without a record of what data means and where sensitive data lives,
  compliance work (PII/SPI handling, audits) has to assume the worst case
  everywhere, which costs more in ongoing overhead than documenting it
  once would have.
- **Schema drift between environments becomes a recurring, manual
  reconciliation problem** when changes aren't tracked through the same
  controlled process as application code changes — needing a schema-diff
  tool to figure out what's different is a symptom that the deployment
  process itself has a gap.
- **The database is the foundation the application is built on** — if it
  doesn't actually solve the project's real requirements, or if no one
  understands it well enough to change it safely, that risk doesn't stay
  contained to "database stuff"; it can force scrapping application work
  built on top of it.

---

## 3. Legitimate Uses

- Genuinely temporary, one-off code — a quick test of an API call, a
  proof-of-concept, a demo for a colleague — doesn't need the full
  documentation/version-control/test treatment. A practical test: if
  you're willing to delete it immediately after use, it's really
  temporary. If you can't bring yourself to delete it, that's a sign it's
  actually worth keeping — and therefore worth committing with at least
  brief notes on what it's for.
- A middle ground for exploratory/ad hoc code that isn't quite "real" but
  also isn't disposable: keep it in a lightweight scratch repository
  (a "gist" or equivalent) rather than either fully engineering it or
  losing it entirely.

---

## 4. The Fix: Apply the Same Discipline to Database Code

Quality assurance has three parts — specify requirements, design/build,
then validate — and database code needs all three exactly as much as
application code does.

### Document the database

At minimum, cover:

- **An ER diagram** (or several, decomposed by subsystem if the schema is
  too large for one diagram to stay readable) showing tables and
  relationships — the single most valuable piece of database
  documentation.
- **Tables, columns, and views**: what entity each table models, what
  each column means (including units for quantitative values), why a
  column is/isn't nullable, expected row volume, expected query patterns.
  A view needs its own rationale — what it abstracts, who's meant to use
  it, whether it's updatable.
- **Relationships**, including the *intent* behind nullability and
  constraints that isn't obvious from the constraint alone — e.g. why is
  `reported_by` required but `assigned_to` optional? What does that imply
  about workflow states? Also document any relationship that's implicit
  (no formal constraint) — without documentation there's no way to
  discover it exists at all.
- **Triggers and stored procedures**: what business rule or problem each
  one exists to solve, documented like an API — inputs, outputs, side
  effects, and why it exists as a trigger/procedure instead of application
  code.
- **SQL security**: what database users/roles exist, what each can access,
  what transport security is required, what protections exist against
  brute-force auth attempts, and whether the code has had a dedicated
  review for [[sql-antipatterns-sql-injection]].
- **Database infrastructure**: brand/version, hostnames, topology
  (replication, clustering, proxies), connection parameters, backup
  policy — information both developers and operators need.
- **ORM-layer logic**: any validation, transformation, caching, or logging
  implemented in ORM model classes is still business logic and needs the
  same documentation as a trigger would.

Documentation that's wrong or stale is worse than none — but the fix for
that is keeping it current as part of the change process, not skipping it.
Even developers who are otherwise skeptical of heavy documentation
overhead tend to agree the database is the one place it's worth the effort,
because schema meaning is very hard to reconstruct after the fact — see
Joel Spolsky's own carve-out for exactly this case, despite being broadly
skeptical of code documentation in general.

### Put database artifacts under the same version control as application code

Check in, alongside application code in the same repository (so a given
revision/tag pulls together a consistent, working set of both):

- **Data definition scripts** (`CREATE TABLE` and friends) for rebuilding
  the schema from scratch.
- **Triggers and stored procedures** — they're part of the application's
  logic, not database "configuration."
- **Bootstrap/seed data** needed to bring a fresh database to a usable
  initial state.
- **ER diagrams and documentation**, kept current as the schema evolves.
- **DBA scripts**: import/export, sync, reporting, backup, validation
  jobs — anything that operates on the database outside the application
  itself.

Use **schema migration tooling** (Rails-style migrations, Alembic,
Liquibase, Flyway, Doctrine, or your framework's equivalent) to manage
schema evolution as a sequence of versioned, reversible steps — an "up"
and a "down" per change — rather than applying ad hoc `ALTER TABLE`
statements by hand and hoping every environment stays in sync:

```ruby
class AddHoursToBugs < ActiveRecord::Migration
  def self.up
    add_column :bugs, :hours, :decimal
  end
  def self.down
    remove_column :bugs, :hours
  end
end
```

This gives you a recorded, reproducible history of schema changes, a way
to bring any environment to a known revision, and a defined path to back
a change out if needed.

### Test the database independently, not just through the application

Apply the same isolation-testing principle used for application code —
test the database layer on its own, so a failure narrows down to the
database rather than somewhere in a larger end-to-end path:

```python
class TestDatabase(unittest.TestCase):
    def test_table_bugs_column_bugid_exists(self):
        self.cursor.execute("SELECT bug_id FROM Bugs LIMIT 1")

    def test_table_bugs_column_issueid_not_exists(self):
        with self.assertRaises(mysql.connector.errors.ProgrammingError):
            self.cursor.execute("SELECT issue_id FROM Bugs LIMIT 1")
```

Cover, as a checklist:

- **Tables/columns/views exist** (and, via negative tests, that removed
  ones are actually gone) — add a test alongside every schema change.
- **Constraints actually reject what they should** — attempt inserts/
  updates that should violate `NOT NULL`, `UNIQUE`, or foreign key
  constraints, and confirm they fail. A constraint that silently allows a
  violation is a bug you want to catch in a test, not in production.
- **Triggers produce their intended side effects** — execute the
  triggering statement, then query to confirm the trigger actually did
  what it was supposed to.
- **Stored procedures**, tested like any function: valid and invalid
  inputs, every logical branch, and any side effects (data changes) they
  cause.
- **Bootstrap/seed data is present** where expected.
- **Queries used by the application** — validate syntax and result shape
  (column names, types) independent of the application code that issues
  them.
- **ORM classes**, tested like the application code they are — including
  that they reject invalid input, not just that they accept valid input.

If a database test fails unexpectedly, treat "wrong environment"
(accidentally pointed at production, a replica, or a not-yet-migrated
database) as a real possibility to rule out before assuming the schema
itself is broken.

### Give every branch/developer their own database instance

A database isn't naturally revision-controlled the way code is, so working
across multiple application branches needs a matching multiplicity of
database instances — one per branch/environment/developer — rather than
everyone sharing a single mutable instance that can't reflect more than
one schema revision at a time. Make the database connection configurable
per environment so switching branches doesn't require hand-editing
connection details. Container/cloud tooling has made spinning up
disposable, production-matching database instances cheap enough that
there's little excuse not to.

---

## 5. Review Checklist

- Is the schema documented anywhere beyond "ask the person who built it"
  — ER diagram, table/column purpose, relationship intent?
- Are schema definitions, triggers, stored procedures, seed data, and
  DBA/ops scripts checked into the same version control as application
  code, or do they only exist live on a database server?
- Are schema changes applied through migration tooling with a recorded,
  reversible history, or through ad hoc manual `ALTER` statements?
- Does the database have its own automated tests (structure, constraints,
  triggers, procedures), independent of application-level tests that only
  exercise it indirectly?
- Does every developer/branch have an isolated database instance to work
  against, or is there contention over a shared one?
- If someone who knows this database left tomorrow, could someone else
  actually pick up where they left off?
