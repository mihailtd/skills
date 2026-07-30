---
name: python-upgrade-dependencies
description: Upgrade project dependencies in pyproject.toml to their latest available versions, resolved against whatever index is actually configured (public PyPI and/or a private registry). Uses uv to find and apply upgrades, pins exact versions with ==, and is registry-aware — never assumes public PyPI is the source of truth for version resolution. Use when updating dependencies, addressing security vulnerabilities, or performing routine maintenance.
---

# Upgrade Project Dependencies Skill

This skill guides you through upgrading Python dependencies using `uv`, ensuring all versions are pinned exactly with `==` for reproducible builds.

It works for both **single-app** projects and **uv workspace monorepos**.

## Overview

The project uses:
- **`uv`** for dependency management (no pip/poetry/pipenv)
- **Exact version pinning**: All dependencies use `==` operator
- **`pyproject.toml`**: Dependency declarations live here

This ensures:
- **Reproducible builds** across all environments
- **Predictable behavior** — no surprise updates
- **Easier debugging** — exact versions in logs
- **Security auditing** — clear inventory of dependencies

## Quick Reference

```bash
# Check current dependency versions
uv pip list

# Find the latest version resolvable against whichever index(es) are
# actually configured for this project (public PyPI, private, or both) —
# see "Private Registries" below for why this matters more than a raw
# PyPI query.
uv add "package-name" --upgrade --dry-run

# Upgrade a single package
uv add "package-name==X.Y.Z"

# Sync environment after manual pyproject.toml edits
uv sync

# Verify installation
uv pip list | grep package-name

# Run tests after upgrade
uv run pytest
```

## Prerequisites

1. **`uv` installed**
2. **Working git branch**: Make changes on a feature branch
3. **Tests passing**: Ensure current state is stable before upgrading
4. **Understand project layout**: Single-app or monorepo (see below)

## Detecting Project Layout

Before upgrading, determine the project layout:

### Single-App Project

One `pyproject.toml` (at root or in a subdirectory like `backend/`), one lock file.

```
project/
├── pyproject.toml       # All dependencies here
├── uv.lock              # Lock file
└── src/
```

### UV Workspace Monorepo

A root `pyproject.toml` with `[tool.uv.workspace]`, multiple member `pyproject.toml` files, one shared `uv.lock`.

```
project/
├── pyproject.toml       # Workspace root (dev deps, workspace config)
├── uv.lock              # Shared lock file
├── .venv/               # Shared virtual environment
└── apps/
    ├── app-a/
    │   └── pyproject.toml
    └── app-b/
        └── pyproject.toml
```

**Detection**:
```bash
# Check for workspace declaration
grep -q "tool.uv.workspace" pyproject.toml && echo "MONOREPO" || echo "SINGLE-APP"
```

## Version Pinning Rules

### DO: Use Exact Version Pinning

```toml
[project]
dependencies = [
    "fastapi==0.115.0",
    "sqlalchemy[asyncio]==2.0.36",
    "uvicorn==0.32.0",
]
```

### DON'T: Use Version Ranges

```toml
[project]
dependencies = [
    "fastapi>=0.110.0",         # ❌ Minimum version
    "sqlalchemy~=2.0.30",       # ❌ Compatible release
    "uvicorn>=0.30.0,<0.33.0",  # ❌ Range
    "pydantic",                 # ❌ No version specified
]
```

## Private Registries

Read this before running any "find latest version" step. A project may resolve
some or all of its dependencies from a private index (an internal Artifactory/
Nexus/CodeArtifact/devpi mirror, or a private PyPI-compatible registry) instead
of, or in addition to, public PyPI. Getting this wrong means either upgrading to
a version that isn't actually installable, or leaking the existence/name of an
internal-only package to a public API while researching it.

### Detecting registry configuration

Check, in order:

1. **`pyproject.toml`** — `[[tool.uv.index]]` entries declare additional indexes:
   ```toml
   [[tool.uv.index]]
   name = "internal"
   url = "https://pkg.internal.example.com/simple"
   explicit = true   # only used when a package is explicitly pinned to it below

   [tool.uv.sources]
   my-internal-lib = { index = "internal" }
   ```
   `explicit = true` means that index is **only** used for packages listed under
   `[tool.uv.sources]` pointing at it — it does not silently become a fallback
   for every package. A package with no `tool.uv.sources` entry resolves from
   the default index chain (public PyPI plus any non-explicit private indexes).
2. **`uv.toml`** (project or user-level) — can declare the same `index`/`sources`
   configuration outside `pyproject.toml`.
3. **Environment variables** — `UV_INDEX_URL`, `UV_EXTRA_INDEX_URL`,
   `UV_INDEX_<NAME>_USERNAME` / `UV_INDEX_<NAME>_PASSWORD` for per-index auth.
4. **`.netrc`** — credential-based auth for an index that doesn't use env-var auth.

### Rules

- **Resolve versions through `uv`, always** (Step 2 below) — never hit
  `pypi.org` directly to decide "what's the latest version." `uv` already
  respects whichever index configuration applies; a raw PyPI query does not,
  and will silently give you a version that may not exist on the configured
  private index (or vice versa: report a package as "not found" when it's
  actually only published privately).
- **Treat any package pinned via `[tool.uv.sources]` to a private/explicit
  index as private** for research purposes (changelogs, GitHub releases — see
  the upgrade-workflow agent). Don't assume its name maps to a public PyPI
  project of the same name; don't try to fetch a public changelog for it
  unless you've independently confirmed it's an internal mirror of a real
  public upstream project.
- **Never print or log credentials.** If you find index auth configured (env
  vars, `.netrc`), note only that auth is configured for that index — never
  the username, password, or token value, even if you can read it.
- A project can mix both: some dependencies public (no `tool.uv.sources`
  entry, resolved from PyPI), others private (explicitly sourced). Handle
  each dependency according to its own resolution, not a single project-wide
  assumption.

## Upgrade Process

### Step 1: Identify Scope

**Single-app**: Just one `pyproject.toml` to update.

**Monorepo**: Check which workspace members use the dependency:

```bash
# Find which members use a dependency
grep -r "package-name" apps/*/pyproject.toml
```

> **Monorepo Rule**: All workspace members that share a dependency **must** use the same version, because they share a single `uv.lock`.

### Step 2: Find Latest Available Versions

**Always resolve through `uv`, not a raw PyPI query.** `uv` already knows which
index(es) this project is configured to use — public PyPI, a private registry,
or both with per-package scoping (see **Private Registries** below). A direct
`pypi.org` query bypasses that entirely: for a private/internal package it will
404 or, worse, silently return metadata for an unrelated public package that
happens to share the name. Only query PyPI directly for research purposes on a
package you've already confirmed is public (e.g. to read its changelog).

```bash
# Dry-run shows what uv would resolve each package to, against the
# configured index(es), without changing pyproject.toml or uv.lock yet.
uv add "fastapi" --upgrade --dry-run
uv add "sqlalchemy" --upgrade --dry-run
uv add "pydantic" --upgrade --dry-run

# Or, to check every dependency in one pass without touching anything:
uv lock --upgrade --dry-run
```

### Step 3: Update Dependency Versions

**Single-app** — use `uv add` or edit `pyproject.toml` directly:

```bash
uv add "fastapi==0.115.0"
```

**Monorepo** — update ALL members that use the dependency simultaneously:

```bash
# Find all occurrences
grep -rn "fastapi==" apps/*/pyproject.toml

# Update all at once (using multi_replace_string_in_file or sed)
```

> **CRITICAL**: Always use `==` to pin exact versions.

### Step 4: Sync and Verify

```bash
# Sync environment (regenerates uv.lock)
uv sync

# Verify installed version
uv pip list | grep package-name

# Check lock file was regenerated
git diff uv.lock
```

### Step 5: Run Tests

```bash
uv run pytest
```

### Step 6: Check for Breaking Changes

1. Visit the package's GitHub releases or changelog
2. Look for BREAKING CHANGES or migration guides
3. Update code if needed

### Step 7: Commit

```bash
git add pyproject.toml uv.lock
git commit -m "chore: upgrade package-name to X.Y.Z"
```

For monorepos, stage all affected `pyproject.toml` files.

## Monorepo-Specific Rules

These apply only when the project uses `[tool.uv.workspace]`:

### Rule 1: All members must use compatible versions

Because all apps share a single lock file, you **cannot** have different pinned versions of the same package:

```toml
# ❌ Will cause resolution failures
apps/app-a/pyproject.toml: "openai==2.15.0"
apps/app-b/pyproject.toml: "openai==1.93.0"
```

```toml
# ✅ All apps use the same version
apps/app-a/pyproject.toml: "openai==1.93.0"
apps/app-b/pyproject.toml: "openai==1.93.0"
```

### Rule 2: Check ALL members before upgrading

```bash
grep -r "package-name" apps/*/pyproject.toml
```

### Rule 3: Update all occurrences simultaneously

Use `multi_replace_string_in_file` or batch edits to update every member at once.

### Resolving Workspace Conflicts

When `uv sync` fails with conflict errors:

1. **Find compatible versions**: check the configured index (via `uv`, not a raw public-registry query) for version constraints of related packages
2. **Remove conflicting extras**: e.g., drop an optional extra that pulls in incompatible transitive deps
3. **Align major versions**: If apps diverged, get everyone on the same major version first

```bash
# Check what a package requires — resolved through uv, so it works whether
# the package comes from public PyPI or a configured private index.
uv pip show package-name
```

## Upgrade Strategies

### Strategy 1: Conservative (Recommended)

Upgrade one dependency at a time, test between each:

```bash
# Upgrade fastapi
uv add "fastapi==0.115.0"
uv run pytest
git add -A && git commit -m "chore: upgrade fastapi to 0.115.0"

# Upgrade sqlalchemy
uv add "sqlalchemy[asyncio]==2.0.36"
uv run pytest
git add -A && git commit -m "chore: upgrade sqlalchemy to 2.0.36"
```

### Strategy 2: Grouped by Ecosystem (Moderate Risk)

Upgrade related packages together:

```bash
# Web framework group
uv add "fastapi==0.115.0" "uvicorn==0.32.0" "pydantic==2.10.0"
uv run pytest
git add -A && git commit -m "chore: upgrade web framework deps"

# Database group
uv add "sqlalchemy[asyncio]==2.0.36" "alembic==1.14.0" "asyncpg==0.30.0"
uv run pytest
git add -A && git commit -m "chore: upgrade database deps"
```

### Strategy 3: Version Check Script

Scan all dependencies and compare current vs. latest **resolvable against the
configured index(es)** — do this through `uv`, not a hand-rolled PyPI client,
for the same private-registry reason as Step 2 above:

```bash
#!/usr/bin/env bash
set -euo pipefail

for pyproject in $(find . -name pyproject.toml); do
  dir=$(dirname "$pyproject")
  echo "== $pyproject =="
  (cd "$dir" && uv lock --upgrade --dry-run) 2>&1 \
    | grep -E "^(Would|Updated|~)" || echo "  (up to date)"
done
```

`uv lock --upgrade --dry-run` reports exactly what would change without
touching `pyproject.toml` or `uv.lock`, and it resolves through whatever
index configuration is actually in effect for that project — public, private,
or a mix. If you need a package-by-package table instead of `uv`'s own diff
output, parse `uv add <pkg> --upgrade --dry-run` per dependency the same way.

## Security Updates

### Check for Vulnerabilities

```bash
uvx pip-audit
```

### Prioritize Security Patches

When vulnerabilities are found:

1. Upgrade affected packages immediately
2. Test thoroughly but quickly
3. Deploy with urgency
4. Document in commit message:

```bash
git commit -m "security: upgrade package-name to fix CVE-XXXX-YYYY

Critical security update. See: https://github.com/package/security/advisories/GHSA-xxxx"
```

## Troubleshooting

### "Could not find a version that satisfies the requirement"

Version doesn't exist or has been yanked:

```bash
uv pip index versions package-name
```

### "Dependency conflict detected"

Two packages require incompatible versions of a shared dependency:

```bash
# Check what depends on the conflicting package
uv pip show conflicting-package

# Try upgrading the dependent packages together
uv add "dependent-a==X.Y.Z" "dependent-b==X.Y.Z"
```

### "Import error after upgrade"

Breaking API changes in new version — downgrade and review the migration guide:

```bash
uv add "package-name==OLD.VERSION"
```

### "Tests fail after upgrade"

API changes or new behavior:

```bash
# Review the package changelog
# Update code to match new API
# Or pin to the previous working version temporarily
uv add "package-name==OLD.VERSION"
```

## Best Practices

1. **Create a feature branch**: `git checkout -b chore/upgrade-dependencies`
2. **Upgrade in small batches**: Don't upgrade everything at once
3. **Read release notes**: Check CHANGELOG for BREAKING / MIGRATION / DEPRECATED keywords
4. **Test thoroughly**: `uv run pytest` after each batch
5. **Update lock file**: `uv.lock` is auto-updated by `uv add`; verify with `uv lock --check`
6. **Document major upgrades**: Note breaking changes in PR description
7. **Monitor after deployment**: Watch error logs, performance metrics, deprecation warnings

## Checklist

Before starting:
- [ ] Create feature branch
- [ ] Ensure tests passing on current versions (skip if there are no tests)
- [ ] Review current dependency list

During upgrade:
- [ ] Search PyPI for latest version of each package
- [ ] Use exact version pinning with `==`
- [ ] (Monorepo) Update ALL workspace members that use the dependency
- [ ] Run tests after each upgrade or batch
- [ ] Check for deprecation warnings
- [ ] Review release notes for breaking changes

After upgrade:
- [ ] All tests passing
- [ ] `uv.lock` is regenerated and committed
- [ ] `pyproject.toml` shows new versions with `==`
- [ ] Application starts without errors
- [ ] Commit with clear message listing upgrades

## Related Resources

- [uv documentation](https://docs.astral.sh/uv/)
- [PyPI - Python Package Index](https://pypi.org/)
- [Python Packaging User Guide](https://packaging.python.org/)
