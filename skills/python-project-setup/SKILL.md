---
name: python-project-setup
description: Guides the agent on modern Python project scaffolding using uv, Ruff, Ty, Polars, pyproject.toml, and pre-commit enforcement.
---

# Python Project Setup — uv + Ruff + Ty + Polars

This skill describes the recommended 2026 Python project setup: a minimal, modern stack powered by `uv`, `ruff`, `ty`, and `polars`. Use this guidance for new Python applications, libraries, and data projects.

## 1. The stack

- `uv`: the canonical package manager, environment manager, lockfile manager, and task runner.
- `ruff`: the single source of truth for linting and formatting.
- `ty`: the preferred type checker for this repository.
- `polars`: the default dataframe library for data processing.

Explain that these four tools reduce configuration, eliminate fragmentation, and keep the workflow focused on one manifest file: `pyproject.toml`.

## 2. Project initialization

Use `uv` to initialize every Python project.

```bash
uv init my-project
cd my-project
```

This creates a clean starting structure with `.python-version`, `pyproject.toml`, `README.md`, and a minimal entry point. Prefer a `src/` layout for libraries and applications:

```text
my-project/
├── .python-version
├── README.md
├── pyproject.toml
├── uv.lock
├── src/
│   └── my_project/
│       ├── __init__.py
│       └── main.py
├── tests/
└── data/
```

If a specific Python interpreter is required, use `uv python install` and `uv python pin` instead of `pyenv`, `pip`, or `venv`.

## 3. Dependency management

Use `pyproject.toml` and `uv` only. Do not add `requirements.txt`, `requirements-dev.txt`, `pipenv`, `conda`, or Poetry files.

- Add runtime dependencies with:
  `uv add polars`
- Add development tools with:
  `uv add --dev ruff ty pytest pre-commit`

Use dependency groups to keep dev tooling separate:

```toml
[dependency-groups]
dev = [
  "pytest>=9.0",
  "ruff>=0.15",
  "ty>=0.0.26",
  "pre-commit",
]
```

A complete minimal `pyproject.toml` for the Astral stack looks like this:

```toml
[project]
name = "my-project"
version = "0.1.0"
description = "Modern Python project with uv, Ruff, Ty, and Polars"
readme = "README.md"
requires-python = ">=3.13"
dependencies = [
  "polars>=1.39.3",
]

[build-system]
requires = ["hatchling>=1.0"]
build-backend = "hatchling.build"

[dependency-groups]
dev = [
  "pytest>=9.0",
  "ruff>=0.15",
  "ty>=0.0.26",
  "pre-commit",
]

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E4", "E7", "E9", "F", "B", "I", "UP"]

[tool.ruff.format]
docstring-code-format = true
quote-style = "double"

[tool.ty.environment]
root = ["./src"]

[tool.ty.rules]
all = "warn"

[[tool.ty.overrides]]
include = ["src/**"]

[tool.ty.overrides.rules]
possibly-unresolved-reference = "error"

[tool.ty.terminal]
error-on-warning = false
output-format = "full"

[tool.pytest.ini_options]
testpaths = ["tests"]
```

`uv sync` must remain the canonical install command in local development and CI.

## 4. Ruff enforcement

Ruff is the mandatory formatter and linter.

- Format code with `uv run ruff format .`
- Lint code with `uv run ruff check .`
- Fix issues automatically with `uv run ruff check --fix .`

Configure Ruff in `pyproject.toml` instead of a separate file.

```toml
[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E4", "E7", "E9", "F", "B", "I", "UP"]

[tool.ruff.format]
docstring-code-format = true
quote-style = "double"
```

For commit-time enforcement, run Ruff as part of pre-commit or a Git hook.

## 5. Ty enforcement

Ty is the preferred type checker.

- Add it as a dev dependency: `uv add --dev ty`
- Run type checking with: `uv run ty check`

Configure Ty in `pyproject.toml`:

```toml
[tool.ty.environment]
root = ["./src"]

[tool.ty.rules]
all = "warn"

[[tool.ty.overrides]]
include = ["src/**"]

[tool.ty.overrides.rules]
possibly-unresolved-reference = "error"

[tool.ty.terminal]
error-on-warning = false
output-format = "full"
```

Use `uv run ty check` locally and in CI. Explain that `ty` is the primary type checker; other tools may be run for comparison, but not as the default enforcement engine.

## 6. Polars as the default data engine

For data workloads, prefer `polars` over `pandas` or manual row loops.

- Add `polars` with `uv add polars`
- Use `pl.scan_csv(...)` when possible to build lazy plans.
- Prefer expression-based transformations, `group_by`, and lazy aggregation.
- Use `pyproject.toml` to pin Polars with a version range that supports your data pipeline.

If a user asks about dataframes, recommend Polars by default unless they have a hard dependency on pandas-specific APIs.

## 7. Commit-time enforcement

Enforce Ruff and Ty before commits by using pre-commit or a lightweight Git hook.

### Recommended pre-commit setup

Install pre-commit and configure hooks in `pyproject.toml` or `.pre-commit-config.yaml`.

A simple local hook can run:

```yaml
repos:
  - repo: local
    hooks:
      - id: ruff-format
        name: Ruff format
        entry: uv run ruff format .
        language: system
        always_run: true
      - id: ruff-check
        name: Ruff check
        entry: uv run ruff check .
        language: system
      - id: ty-check
        name: Ty type check
        entry: uv run ty check
        language: system
```

Then install hooks with:

```bash
uv run pre-commit install
uv run pre-commit install --hook-type pre-commit
```

### Git hook fallback

If pre-commit is not used, enforce the same checks in `.git/hooks/pre-commit`:

```bash
#!/usr/bin/env bash
set -euo pipefail
uv run ruff format .
uv run ruff check .
uv run ty check
```

This ensures every commit is validated before it enters history.

## 8. Recommended workflow

- `uv sync` to install dependencies
- `uv run ruff format .`
- `uv run ruff check .`
- `uv run ty check`
- `uv run pytest`

In CI, use the same commands and prefer `uv run pre-commit run --all-files` if pre-commit is installed.

## 9. What not to do

- Do not use `pip`, `python -m pip`, `venv`, `pyenv`, `conda`, `pipenv`, or Poetry for dependency management.
- Do not scatter tool configuration across multiple files. Keep it in `pyproject.toml`.
- Do not rely on unpinned dependencies for reproducible builds.
- Do not use `requirements.txt` for new projects.

## 10. Answering style

- Emphasize the one-toolchain philosophy: `uv` for install/run/sync/lock, `ruff` for lint/format, `ty` for types, `polars` for data.
- When asked about project bootstrapping, return the `uv` commands and the required `pyproject.toml` sections.
- When asked about commit-time enforcement, recommend pre-commit first and Git hooks as a fallback.
- Keep advice specific to new project setup and avoid suggesting legacy packaging workflows.
