---
name: python-notebooks-marimo
description: Instructs the agent to use marimo instead of Jupyter for Python notebooks — reactive execution that eliminates hidden state and out-of-order bugs, storage as plain, git-diffable .py files instead of JSON .ipynb blobs, instant conversion to interactive web apps, and script/CLI execution. Also covers marimo pair, marimo's own AI-agent pairing skill for live in-notebook collaboration. Use when creating a notebook, reviewing one, or explaining why this project doesn't use Jupyter.
---

# Python Notebooks — marimo over Jupyter

This is a team standard: **use marimo, not Jupyter, for all Python
notebooks in this project.** This skill explains why, what it changes about
how notebooks are written and reviewed, and how to use marimo's own
AI-pairing capability (`marimo pair`) for live, in-notebook agent
collaboration.

---

## When to use this skill

Use this skill when you need to:

- create a new Python notebook for exploration, analysis, or a
  demo/dashboard,
- review a notebook (`.py` or `.ipynb`) submitted as part of a change,
- explain to a teammate, new hire, or stakeholder why this project doesn't
  use Jupyter,
- convert an existing Jupyter notebook into the project's standard format,
- pair-program interactively inside a live notebook session using an AI
  coding agent.

---

## Outcome

Produce notebook work that:

- is authored and stored as a marimo notebook (`.py` file), never as a
  Jupyter `.ipynb` file, for anything committed to this project,
- has no cell that depends on execution order — the same top-to-bottom run
  always produces the same result,
- diffs cleanly in version control, since the file is plain, readable
  Python rather than a JSON blob carrying execution counts and cell
  metadata,
- can run as a normal script or CLI, or serve as an interactive app, without
  restructuring the underlying code,
- when AI-assisted, uses marimo pair's live, in-memory access rather than
  an assistant that can only see static file contents.

---

## Instructions for the AI

1. **Default to marimo for every notebook in this project**
   - When asked to create a notebook, scaffold it as a marimo notebook
     (`.py` file, `import marimo` + `@app.cell` structure), not a Jupyter
     `.ipynb` file.
   - If an existing `.ipynb` file is encountered in this project (e.g., in
     a PR, or inherited from elsewhere), recommend converting it to marimo
     rather than extending it in place — flag `.ipynb` files during review
     as a deviation from the team standard, not a stylistic preference to
     leave alone.
   - Install with `uv add marimo` in projects already using `uv` (see
     python-project-setup), consistent with the project's standard package
     manager.

2. **Explain the justification in concrete, not just preferential, terms**
   - **Reactive execution:** when a cell's inputs change, marimo
     automatically re-runs every cell that depends on that value. This
     directly eliminates the classic Jupyter failure mode of stale outputs
     from out-of-order execution — a cell's displayed output is guaranteed
     to reflect its current inputs, not whatever was run last in whatever
     order.
   - **No hidden state:** marimo enforces that each variable is defined in
     exactly one cell and disallows the kind of implicit, order-dependent
     state Jupyter permits. This guarantees the notebook produces the same
     result whether run top-to-bottom fresh or interactively cell-by-cell —
     a guarantee Jupyter notebooks structurally cannot make.
   - **Git-friendly storage:** a marimo notebook is a plain `.py` file, so
     `git diff` shows actual code changes. A Jupyter `.ipynb` file is a JSON
     document carrying cell execution counts, output blobs, and metadata
     alongside the code, which makes diffs noisy and merge conflicts far
     more painful — this is a genuine version-control quality improvement,
     not just a taste preference.
   - **Script and CLI ready:** because a marimo notebook is a normal Python
     file, it can be executed directly (`python notebook.py` or `marimo run
     notebook.py`) without any notebook-specific export or conversion step
     — the same file works as an exploratory notebook and as a runnable
     script.
   - **Built-in apps and UI:** marimo can turn notebook variables and code
     blocks into an interactive web app or dashboard using its built-in UI
     elements, without a separate app-building framework or rewrite.
   - When asked to justify the standard (to a teammate, in a doc, or in a
     PR comment), lead with reactive execution and no-hidden-state, since
     these are the two properties that directly prevent a real, common
     class of notebook bugs — the git-friendliness and app/script
     flexibility are strong secondary reasons.

3. **Write marimo notebooks idiomatically, not like disguised Jupyter cells**
   - Respect marimo's single-definition-per-variable constraint — don't
     work around it by stuffing logic into one giant cell just to avoid
     restructuring; instead, decompose the notebook into cells organized
     around genuinely distinct steps, letting the reactive graph do the
     dependency tracking.
   - Prefer marimo's built-in UI elements (`mo.ui.*`) for interactive
     parameters over hardcoded values when a notebook is meant to be
     explored interactively or shared as a lightweight app.
   - When a notebook is meant to also run as a script (e.g., in CI or a
     scheduled job), keep side-effecting or expensive cells guarded
     appropriately and verify the notebook runs cleanly via `marimo run` or
     as a plain script, not just interactively in the editor.

4. **Use marimo pair for live, in-notebook AI collaboration**
   - Explain marimo pair as marimo's own AI-agent pairing capability,
     distinct from a general-purpose coding assistant: it gives a
     coding-agent tool (e.g., Claude Code) read-and-write access to the
     *live* notebook session — active Python variables, dataframes, and
     outputs in memory — not just the static `.py` file on disk.
   - Note the concrete capabilities this enables: the agent can run cells,
     edit or delete broken logic, and install missing packages
     dynamically, all within the live session — and, because marimo
     notebooks are reactive dataflow graphs, any cell the agent adds or
     changes automatically triggers its downstream dependents to re-run,
     which is what prevents the agent from silently leaving the notebook in
     a broken, inconsistent state.
   - Note the scratchpad behavior: the agent can test hypotheses in an
     invisible background scratchpad and only commit changes to the
     primary notebook once the logic is confirmed working — recommend
     treating this as the expected, safe default workflow when pairing,
     rather than having the agent edit primary cells experimentally.
   - Installation and usage:
     ```bash
     # via npx / node
     npx skills add marimo-team/marimo-pair
     # via uv / python
     uvx deno -A npm:skills add marimo-team/marimo-pair

     # then, inside a coding-agent session:
     /marimo-pair pair with me on my_notebook.py
     ```
   - Recommend marimo pair specifically for live, exploratory,
     back-and-forth notebook work — for one-shot notebook creation or
     scripted edits to a notebook file, standard file-based editing (Read/
     Edit/Write on the `.py` file) is simpler and doesn't require the live
     session to be running.

---

## Decision points and guidance

- **Is this a new notebook, or an inherited `.ipynb`?** New notebooks
  always start as marimo; inherited Jupyter notebooks should be flagged and
  converted rather than extended in place.
- **Does the notebook need to run outside an interactive session (CI,
  scheduled job, CLI)?** If so, lean on marimo's script/CLI-readiness
  rather than building a separate export step.
- **Is the AI collaboration one-shot or live/iterative?** Live, exploratory
  pairing benefits from `marimo pair`'s in-memory access; a single scripted
  edit doesn't need it.
- **Is a cell trying to redefine a variable already defined elsewhere?**
  Treat this as a signal to restructure the notebook's cell boundaries, not
  to bypass marimo's single-definition rule.

---

## Quality criteria

A strong marimo notebook should ensure that:

- **it's a `.py` file**, never a `.ipynb`, for anything committed to this
  project,
- **execution order doesn't matter:** a fresh top-to-bottom run produces
  the same result as the current interactive state,
- **diffs are clean:** changes to the notebook show up as readable code
  diffs, not JSON metadata churn,
- **it runs standalone:** the notebook can execute as a script or via
  `marimo run` without modification,
- **AI pairing sessions use the scratchpad workflow** for experimentation,
  only committing confirmed-working changes to the primary notebook.

---

## Example prompts

- "Set up a new marimo notebook for exploring this dataset."
- "This PR includes a Jupyter `.ipynb` file — help me convert it to
  marimo."
- "Why does our team use marimo instead of Jupyter? I need to explain this
  to a new teammate."
- "Pair with me on this notebook using marimo pair to test a few approaches
  before committing to one."

---

## Related guidance

Use this tool alongside:

- python-project-setup
- python
- python-polars-eager-api
- database-duckdb-marimo-notebooks (package `database-duckdb`) — writing DuckDB SQL cells (`mo.sql()`), custom connections, and UI-parameterized queries inside a marimo notebook
