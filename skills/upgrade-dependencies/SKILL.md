---
name: upgrade-dependencies
description: Upgrade project dependencies in pyproject.toml to their latest versions from PyPI. Searches for latest versions, uses uv to upgrade, and pins exact versions with ==. Use when updating dependencies, addressing security vulnerabilities, or performing routine maintenance.
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

# Search PyPI for latest version
python3 -c "import urllib.request, json; data = json.loads(urllib.request.urlopen('https://pypi.org/pypi/PACKAGE/json').read()); print('Latest:', data['info']['version'])"

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

## Upgrade Process

### Step 1: Identify Scope

**Single-app**: Just one `pyproject.toml` to update.

**Monorepo**: Check which workspace members use the dependency:

```bash
# Find which members use a dependency
grep -r "package-name" apps/*/pyproject.toml
```

> **Monorepo Rule**: All workspace members that share a dependency **must** use the same version, because they share a single `uv.lock`.

### Step 2: Find Latest Versions on PyPI

```bash
python3 << 'EOF'
import json
import urllib.request

packages = ["fastapi", "sqlalchemy", "pydantic"]
for pkg in packages:
    url = f"https://pypi.org/pypi/{pkg}/json"
    try:
        with urllib.request.urlopen(url, timeout=5) as response:
            data = json.loads(response.read())
            latest = data['info']['version']
            print(f"{pkg:30} -> {latest}")
    except Exception as e:
        print(f"{pkg:30} -> ERROR: {str(e)[:50]}")
EOF
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

1. **Find compatible versions**: Check PyPI for version constraints of related packages
2. **Remove conflicting extras**: e.g., drop an optional extra that pulls in incompatible transitive deps
3. **Align major versions**: If apps diverged, get everyone on the same major version first

```bash
# Check what a package requires
python3 -c "
import urllib.request, json
data = json.loads(urllib.request.urlopen('https://pypi.org/pypi/PACKAGE/json').read())
print([r for r in data['info']['requires_dist'] or [] if 'CONFLICT' in r.lower()])
"
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

Scan all dependencies and compare against PyPI:

```python
#!/usr/bin/env python3
import json
import re
import urllib.request
from pathlib import Path

# Find all pyproject.toml files
pyproject_files = list(Path(".").rglob("pyproject.toml"))
all_deps: dict[str, set[str]] = {}

for pyproject in pyproject_files:
    content = pyproject.read_text()
    for match in re.finditer(r'"([a-z0-9_-]+)\[?[a-z,]*\]?==([0-9.]+)"', content, re.IGNORECASE):
        pkg, version = match.groups()
        all_deps.setdefault(pkg, set()).add(version)

print(f"{'Package':<30} {'Current':<15} {'Latest':<15} {'Status'}")
print("=" * 75)

for pkg, versions in sorted(all_deps.items()):
    try:
        url = f"https://pypi.org/pypi/{pkg}/json"
        with urllib.request.urlopen(url, timeout=5) as response:
            data = json.loads(response.read())
            latest = data["info"]["version"]
            for v in sorted(versions):
                status = "✅" if v == latest else "⚠️  upgrade"
                print(f"{pkg:<30} {v:<15} {latest:<15} {status}")
    except Exception:
        print(f"{pkg:<30} {'?':<15} {'ERROR':<15}")
```

## Security Updates

### Check for Vulnerabilities

```bash
pip install pip-audit
pip-audit
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
- [ ] Ensure tests passing on current versions
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
