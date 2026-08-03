---
name: sql-antipatterns-standard-operating-procedures

description: Detects the Standard Operating Procedures (Follow the Leader) antipattern — defaulting to stored procedures for all database logic "because that's how it's always been done," without weighing it against the project's actual architecture — which concentrates load and business logic on a single non-scalable database server, is hard to develop/test/deploy safely, and isn't portable across brands. Covers MySQL-specific stored procedure limitations in depth. Guides toward evaluating stored procedures case by case against a modern, horizontally-scaled application architecture.
---

# SQL Antipatterns — Standard Operating Procedures

This skill helps evaluate whether stored procedures are the right choice
for a given piece of logic, instead of applying them as a blanket rule
inherited from a past project or a past era of software engineering — and
covers MySQL's specific stored procedure limitations for teams considering
them there.

---

## 1. Recognize the Antipattern

- **The tell:** stored procedures used for most or all database logic as a
  fixed policy, justified by precedent rather than by evaluating this
  project's actual architecture:

  > "We decided at the start of the project that all SQL code should be
  > implemented in stored procedures." / "We were told that's the way to
  > develop database applications."

- **The antipattern isn't stored procedures themselves** — it's adopting
  any technology choice by inherited habit ("follow the leader") instead
  of deliberately evaluating it against the current project's
  architecture, team skills, and scaling needs. Stored procedures are just
  the chapter's example; the same failure mode applies to any tool chosen
  because "that's how we've always done it."
- **Listen for these phrases**:
  - "We always use procedures because they give better performance." — an
    absolute claim about a tool whose performance impact is genuinely
    case-by-case; treat categorical performance claims about any single
    feature with suspicion.
  - "What tool can I use to migrate 500+ stored procedures from T-SQL to
    MySQL?" — assumes automatic cross-brand procedure translation exists;
    it doesn't, because procedure languages aren't portable (see §2).
  - Confusion over brand-specific procedure syntax (e.g. MySQL's `DECLARE`
    only valid at the top of the main block, no `@` prefix on local
    variables) — a sign of assuming one brand's procedure dialect
    transfers to another.

---

## 2. Why Defaulting to Stored Procedures Breaks Down

- **Not portable.** Despite a standard existing on paper, every brand
  implements stored procedure syntax differently — switching database
  brands means rewriting every procedure by hand, re-deriving the intended
  logic each time, not a mechanical translation.
- **Weaker tooling than application code gets.** Most mainstream IDEs and
  debuggers (breakpoints, variable inspection) don't support stored
  procedure languages the way they support Java, Python, or Go — you're
  pushed toward specialized, often clunkier, brand-specific tools.
- **Deployment is a different, harder problem than deploying application
  code.** Application servers can be restarted one at a time behind a load
  balancer for zero-downtime deploys. A database server is typically a
  single instance — replacing a procedure's code (`CREATE OR REPLACE
  PROCEDURE`, or brand equivalents) has to wait for in-flight calls to
  finish and blocks new calls meanwhile. A slow-running procedure can make
  this deployment pause long enough to look like the whole database has
  locked up. Migration tooling (see
  [[sql-antipatterns-diplomatic-immunity]]) also generally treats stored
  procedures as an afterthought — an opaque SQL script to run, without the
  structured up/down support it gives tables and columns.
- **Concentrates load on the one server that can't scale out
  horizontally.** Procedure code runs on the database server, competing
  for the same CPU that's doing query execution. Meanwhile application
  servers — which usually *can* scale out — sit idle waiting for
  procedure calls to return. The database server becomes the bottleneck
  for the whole system precisely because it's the one tier you can't just
  add more of.
- **Compilation behavior varies and is easy to get wrong.** Some brands
  cache a compiled form of a procedure; some don't. Some report syntax
  errors immediately on `CREATE PROCEDURE`; others only at first execution
  — meaning a broken procedure can sit undetected until it's actually
  called, possibly in production.
- **Scalability past a single server is awkward.** A stored procedure
  executes on one database server and can only directly touch data on
  that server. Reaching data on another shard/server needs
  brand-specific cross-server features (foreign data wrappers, database
  links, linked servers), which add configuration overhead, query/
  transaction limitations, and another network dependency that can fail.

---

## 3. Legitimate Uses

Stored procedures are a genuinely good fit in specific, identifiable
cases — evaluate case by case rather than by blanket policy in either
direction:

- **High client-to-database network latency combined with a task that
  needs several sequential queries.** Running the whole sequence inside
  one procedure call eliminates the round-trip latency between each step.
- **Infrequent tasks with no client application driving them** —
  administrative jobs like privilege audits, cache/log cleanup, scheduled
  maintenance — where there's no application tier to place the logic in
  anyway.
- **Encapsulating operations that need elevated privileges.** Grant the
  procedure itself the sensitive privilege, and grant users only the
  privilege to call the procedure — this lets users self-service a
  prescribed sensitive operation without being granted broad direct table
  access. Handling PII/SPI inside a procedure also avoids that data
  crossing the network to a client and back.
- **Mature, feature-rich database platforms** (Oracle, SQL Server, DB2)
  have more capable procedure environments than the general case implies
  — the calculus is meaningfully different than on a database whose
  procedure support is comparatively bare (see §4 for MySQL specifically).

---

## 4. MySQL-Specific Limitations

If you're specifically weighing stored procedures on MySQL, its
implementation carries extra costs beyond the general case above — MySQL
introduced stored procedures in 5.0 (2005), and most of its ecosystem
historically preferred running queries from application/ORM code instead,
so the procedure tooling never matured the way it did on other brands:

- **No packages/modules/OOP features** — organizing or deploying a large
  collection of procedures has no structural support; it's flat and
  unorganized by default.
- **No debugger support at all** — no breakpoints, no variable inspection.
  Workarounds (inserting logging statements into procedure code) require
  modifying the procedure itself, affecting every caller while debugging
  is in progress.
- **No real unit-testing support.** There's no way to run a MySQL
  procedure in an isolated/mocked environment — testing means calling it
  against a real MySQL instance with real data, which is closer to system
  testing than unit testing. Any true isolation-testing tooling has to be
  built yourself.
- **No cross-session compiled cache.** Each session compiles a procedure
  on first use, and that compiled form is discarded when the session ends
  — no other session benefits from it. For short-lived sessions (typical
  of web applications making brief per-request connections), this
  compilation overhead recurs constantly instead of being paid once.
- **Deploying a procedure change risks real downtime.** There's no atomic
  "replace" — updating a procedure means `DROP PROCEDURE` followed by
  `CREATE PROCEDURE`, and between those two statements any call to the
  procedure fails outright. For a procedure called on every request, this
  can mean many failed calls during deployment, not just a rare unlucky
  one.
- **Sharded architectures are awkward.** MySQL's cross-server table access
  requires the `FEDERATED` storage engine, which is limited. Practical
  workarounds are clunky: either the client must already know which shard
  holds the data it needs and connect directly to that server's session,
  or it must call the same procedure across every shard and merge results
  (including empty ones from shards with no matching data). Popular
  sharding middleware built for MySQL (e.g. Vitess) is generally designed
  around plain table queries and may not support stored procedures against
  sharded keyspaces at all.

Given these, the general recommendation for MySQL specifically is to favor
running individual queries from application code over stored procedures,
reserving procedures for the narrow latency-sensitive case in §3 (and even
then, prefer colocating the application with the database on a fast network
over reaching for procedures to hide latency).

---

## 5. The Fix: Evaluate Architecture Deliberately, Per Project

- Default to implementing logic in application code (Java, Python, Go, or
  whatever your team is actually strongest in), where modern editors,
  debuggers, and test frameworks (including mocking) are mature and your
  team is likely more productive than in an unfamiliar procedure language.
- Deploy application logic across multiple horizontally-scaled servers
  behind a load balancer (rolling or blue/green deploys), so load that
  would otherwise concentrate on a single database server gets spread
  across tiers that can actually scale out. The database server still runs
  the SQL queries themselves — the win is moving everything *around* those
  queries (building them, processing results) off the one server that
  can't scale horizontally.
- When a specific case matches one of the legitimate uses in §3, choose
  stored procedures deliberately for that case — don't let a general
  preference for application-tier logic become its own unconditional rule
  either. The point is evaluating each case on its merits, not swapping
  one blanket policy for its opposite.

---

## 6. Review Checklist

- Is "we use stored procedures for everything" (or "we never use them") a
  policy inherited from a prior project/person, or a decision made against
  this project's actual architecture and team skills?
- For logic currently in a stored procedure, does it fit one of the
  legitimate cases in §3 (latency-bound multi-step sequence, no-client
  scheduled task, privilege encapsulation) — or would it scale and deploy
  better as application code?
- If deploying on MySQL specifically, has the team accounted for the lack
  of atomic procedure replacement, the lack of debugger/unit-test support,
  and the per-session recompilation cost?
- Is the database server measurably the bottleneck (per
  [[sql-antipatterns-index-shotgun]]'s Measure/Explain approach) while
  application servers sit idle — a concrete sign that logic concentrated
  in stored procedures should move to the application tier?
- Does deploying a procedure change carry a real risk of visible downtime
  or failed calls, and if so, is that risk actually accounted for in the
  deployment process?
