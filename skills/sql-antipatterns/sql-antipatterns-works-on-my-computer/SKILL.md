---
name: sql-antipatterns-works-on-my-computer

description: Flags dev/production database version and configuration drift — code that relies on a newer SQL feature (syntax, functions, data types) than production runs, working locally and failing on deploy — and recommends matching database versions and settings across environments.
---

# SQL Antipatterns — It Works on My Computer

This skill helps catch a specific, deploy-time-only failure mode: SQL that
depends on a database version, feature, or configuration that exists in
development but not in production.

---

## 1. Recognize the Antipattern

- A query or feature works locally but fails only after deployment, often
  with a syntax error that makes no sense given the code is unchanged:

  ```
  Error: 1064 (42000): You have an error in your SQL syntax; check the
  manual that corresponds to your MySQL server version for the right
  syntax to use near 'WITH'
  ```

- Root cause: the developer's local database is a newer version than
  production (here, `WITH RECURSIVE` requires MySQL 8.0+; the developer's
  machine has it, the production server doesn't).
- Applies to any version-gated capability: recursive CTEs, window function
  variants, JSON functions, newer data types, changed default settings.

---

## 2. Why It Happens

- It's easier to upgrade a database on a laptop than to risk downtime
  upgrading a production server, so local environments drift ahead.
- Application language/library/framework versions are usually pinned and
  installed as part of the build, so they stay in sync automatically —
  database server version is not typically part of that same deployable
  package, even with containerized app deployment, so it's easy to forget
  it needs the same discipline.
- The failure is invisible until deploy time: nothing in code review or
  local testing catches a feature gap between two live database servers.

---

## 3. The Fix: Match Environments, Not Just Versions

- Pin development and production to the same database **version**, as
  close to exact as practical — even minor releases can add features,
  fix bugs, or break backward compatibility.
- Match configuration too, not just the version number:
  - Settings that affect data treatment or query behavior.
  - Global defaults (SQL modes, locale, timezone handling).
  - Users, roles, and privileges — mirror the access structure (but never
    reuse the same passwords/secrets across environments).
- Treat the database server version as a tracked dependency of the
  project, the same way you'd pin a language runtime or library version —
  document it, and check it as part of environment setup, not just app
  library installs.

---

## 4. Review Checklist

- Does a query use a feature (syntax, function, data type) newer than what
  production actually runs? Check the minimum version production supports,
  not just what's installed locally.
- Is the database version pinned and documented anywhere, the way the
  language runtime and dependencies are?
- Do local and production database configurations (SQL mode, timezone,
  locale, privilege structure) match, or only "probably" match?
- If containerized, is the database's version part of that same
  reproducible build, or is it a separately managed service that can drift?
