---
name: database-duckdb-huggingface-datasets

description: Guides the agent through querying Hugging Face-hosted datasets directly from DuckDB via hf:// paths — public datasets, files nested in a folder, multi-file glob loading, persisting a remote dataset into a local table, and the two ways to authenticate against a private dataset (CREATE SECRET with CONFIG vs CREDENTIAL_CHAIN) including which one is safe to put in committed code.
---

# DuckDB and Hugging Face Datasets

Hugging Face hosts a large public (and private) repository of datasets. DuckDB can
query them directly with an `hf://` path — the same remote-querying mechanics as
**database-duckdb-httpfs**, specialized to Hugging Face's dataset-repository
layout.

## 1. The `hf://` Path Scheme

A dataset's files are addressed as `hf://datasets/<owner>/<dataset>/<file>`:

```python
conn.sql("SELECT * FROM 'hf://datasets/scikit-learn/tips/tips.csv'").pl()
```

This works the moment `httpfs` is available — since `httpfs` autoloads (see
**database-duckdb-httpfs** section 1), no separate extension install is required
for `hf://` specifically.

## 2. Persisting to a Local Table

Every query against an `hf://` path re-fetches from the remote endpoint. For
anything queried more than once in a session, materialize it into a local table
first — the same "scan once, work locally after" tradeoff as
**database-duckdb-dataframe-integration** section 4:

```python
conn.execute("CREATE TABLE tips AS FROM 'hf://datasets/scikit-learn/tips/tips.csv'")
conn.sql("SELECT * FROM tips").pl()
```

## 3. Files Nested in a Folder

Some dataset repositories organize files under a subfolder — include it in the
path exactly as shown in the repository's Files tab:

```python
conn.sql("""
    SELECT * FROM
    'hf://datasets/AiresPucrs/adult-census-income/data/train-00000-of-00001-7e70ed54d8cbb057.parquet'
""").pl()
```

## 4. Multiple Files via Glob

The same glob wildcards from **database-duckdb-json-io** section 5 (`*`, `**`,
`?`, `[abc]`, `[a-z]`) work against `hf://` paths too:

```python
conn.sql("SELECT * FROM 'hf://datasets/example/city_country/*.csv'").pl()
```

Every file matched by the glob must share the same schema — a mismatch raises an
error rather than silently producing a partial or malformed result, which is
stricter (and safer) than the silent type-widening behavior called out for local
multi-file JSON loads in **database-duckdb-json-io** section 5.

## 5. Private Datasets — Authentication

A private dataset needs credentials before DuckDB can read it, created with
`CREATE SECRET` — DuckDB's general secrets mechanism, also used for S3/Azure
credentials, not something Hugging-Face-specific.

**`CONFIG` provider** — the token is a literal string in the statement:

```python
conn.execute("""
    CREATE SECRET hf_token (TYPE HUGGINGFACE, TOKEN '{token}')
""".format(token=os.environ["HF_TOKEN"]))
```

**Never hardcode the token as a string literal in committed code** — pull it from
an environment variable or secrets manager (as above), the same rule this
repository applies to every other credential. A token pasted directly into a
notebook or script is one accidental commit away from being a leaked secret.

**`CREDENTIAL_CHAIN` provider** — no token appears in code at all; DuckDB reads
it from a location `huggingface-cli login` already wrote to (typically
`~/.cache/huggingface/token`), the same pattern as picking up cloud credentials
from a local credentials file or environment:

```python
conn.execute("CREATE SECRET hf_token (TYPE HUGGINGFACE, PROVIDER CREDENTIAL_CHAIN)")
```

Prefer `CREDENTIAL_CHAIN` by default — it's the only one of the two that keeps
the token out of source entirely. Reach for `CONFIG` (with the token sourced from
an environment variable, never a literal) only when a specific, explicit token
needs to be supplied per-run rather than picked up from the ambient environment —
e.g. a CI job injecting a scoped token as a secret-manager-provided environment
variable.

## Related guidance

- **database-duckdb-httpfs** — the general remote-file mechanics (`httpfs`, autoloading) this skill's `hf://` scheme builds on.
- **database-duckdb-json-io** — the glob wildcards reused in section 4, and the multi-file schema-consistency concern.
- **database-duckdb-dataframe-integration** — the scan-once-vs-materialize decision behind section 2's advice to persist a remote dataset locally.
- **database-duckdb-external-databases** — DuckDB's other "attach a remote source" pattern (a live PostgreSQL server), for contrast.
