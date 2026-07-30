---
name: python-library-creation
description: >
  Guides the agent and team on building a modern Python library in 2026.
  Covers `uv` initialization, src-layout packaging, Astral tooling, type checking
  with `ty`, testing, CI, pre-commit, and publishing.
---

# Python Library Creation — Modern Best Practices

This skill describes how to build and maintain a Python library in 2026 using the
modern Astral stack: `uv`, `ruff`, and `ty`. It assumes all projects in this
repository enforce `uv` over `pip`, use `pyproject.toml` only, and rely on the
Astral tooling chain for linting, formatting, and type checking.

---

## Initialize Your Project

Start with `uv` as the project initializer. `uv` is the canonical package manager,
build tool, and task runner for this repository.

```bash
uv init --lib my-package
```

This command creates a minimal library scaffold with the recommended `src/`
layout and a `pyproject.toml` manifest. Use normalized package names in lower
skewer case, and name the repository directory to match the package name.

### Recommended layout

.
├── .git
├── .gitignore
├── pyproject.toml
├── .python-version
├── README.md
├── src
│   └── my_package
│       ├── __init__.py
│       └── py.typed
└── tests
    ├── unit
    └── integration

---

## Packaging and Manifest

`pyproject.toml` is the single source of truth for package metadata, dependency
management, build configuration, and tooling settings.

- Do not create or maintain `requirements.txt`, `requirements-dev.txt`, or any
  `requirements/` directory.
- Do not use `pip`, `python -m pip`, `poetry`, or `pipenv` directly.
- Express all dependencies and dev dependencies in `pyproject.toml`.
- Keep `project.version`, `project.description`, `project.authors`,
  `project.keywords`, and `project.classifiers` accurate for publishing.

Use `uv version --bump {major,minor,patch}` to manage releases reliably.

---

## Source Layout

Use the `src/` layout for library code.

- Put package code in `src/<package_name>/`.
- Keep `src/<package_name>/__init__.py` lightweight.
- Export the public API intentionally using `__all__` where appropriate.
- Add `src/<package_name>/py.typed` to signal typing support.

The `src/` layout prevents accidental imports from the repository root during tests
and makes packaging behavior consistent.

---

## Linting and Formatting

The Astral stack is mandatory.

- Add `ruff` as a dev dependency: `uv add --dev ruff`
- Format code with: `uv run ruff format .`
- Enforce linting with: `uv run ruff check .`
- Fix lint issues automatically with: `uv run ruff check --fix .`

Use `ruff` for both linting and formatting so style and quality are enforced by a
single toolchain.

---

## Type Checking

`ty` is the preferred type checker for Python code in this repository.

- Add `ty` as a dev dependency: `uv add --dev ty`
- Run checking with: `uv run ty check`
- If needed, compare against other tools with:
  `uv run --with mypy mypy src/` or `uv run --with pyrefly pyrefly check`.

`ty` should be the primary type checker enforced in CI and documentation. Type
annotations are mandatory for public APIs, internal functions, and test helpers.

Include `uv run ty check` in the commit-time validation flow alongside Ruff.

---

## Testing

Tests belong under a top-level `tests/` folder, typically split into `tests/unit`
and `tests/integration`.

- Use `pytest` as the default runner.
- Add coverage support with `pytest-cov`.
- Keep tests fast, isolated, and deterministic.

Example commands:

```bash
uv add --dev pytest pytest-cov
uv run pytest --cov=my_package --cov-report=term-missing tests/unit
uv run pytest tests/unit tests/integration
```

---

## Supporting Multiple Python Versions

Use `uv` to work with multiple Python interpreters without adding extra tools.

```bash
uv python find 3.12 || uv python install 3.12
uv run --python 3.12 pytest tests/unit
uv python find 3.13 || uv python install 3.13
uv run --python 3.13 pytest tests/unit
```

This is especially useful for libraries that need to remain compatible across
supported interpreter versions.

---

## Maintaining Code Quality

Quality must be enforced both locally and in CI.

### CI

At a minimum, CI should run:

- `uv sync`
- `uv run ruff check`
- `uv run ruff format --check`
- `uv run ty check`
- `uv run pytest tests/`

For libraries that publish releases, also validate package metadata and build
artifacts before publishing.

### Pre-commit

Use `pre-commit` to keep Ruff and Ty enforcement in the commit path.

```bash
uv add --dev pre-commit
uv run pre-commit sample-config
uv run pre-commit install --hook-type pre-commit
```

A minimal `pre-commit` config can run the Astral toolchain directly:

```yaml
repos:
  - repo: local
    hooks:
      - id: ruff-format
        name: Ruff format
        entry: uv run ruff format .
        language: system
        types: [python]
      - id: ruff-check
        name: Ruff check
        entry: uv run ruff check .
        language: system
        types: [python]
      - id: ty-check
        name: Ty type check
        entry: uv run ty check
        language: system
        types: [python]
```

Install and run the hooks with:

```bash
uv run pre-commit install
uv run pre-commit run --all-files
```

If pre-commit is not available, enforce the same checks in a Git hook at
`.git/hooks/pre-commit`:

```bash
#!/usr/bin/env bash
set -euo pipefail
uv run ruff format .
uv run ruff check .
uv run ty check
```

This makes Ruff linting/formatting and Ty type checking a guardrail before every
commit.

---

## Publishing

`uv` is the build and publish tool.

- Use `uv build` to create a distributable package.
- Use `uv publish` to publish to a registry if allowed.
- For internal use, install from source with `uv add /path/to/my-package`.

If publishing to PyPI is not allowed, keep the repository installable from source
with local `uv add` workflows.

---

## The Final Product

A modern Python library repository should include:

- `.git/hooks/pre-commit` or `.pre-commit-config.yaml`
- `src/<package_name>/` with a clean public API
- `tests/` with unit and integration coverage
- `pyproject.toml` as the only manifest
- `Makefile`, `justfile`, or documented scripts for common workflows
- CI workflows that enforce `uv`, `ruff`, `ty`, and `pytest`

This setup keeps package development simple, consistent, and safe.
