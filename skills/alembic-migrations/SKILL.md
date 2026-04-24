---
name: alembic-migrations
description: Create and manage Alembic database migrations for PostgreSQL schema changes. Includes critical rules for immutability, idempotency, and Row Level Security (RLS) enforcement. Use when creating new migrations, modifying schema, or working with database changes.
---

# Alembic Database Migrations Skill

This skill provides comprehensive guidance for creating and managing Alembic database migrations, with special emphasis on security, immutability, and idempotency.

## Quick Reference

**Location**: `backend/alembic/`

**Common Commands**:
```bash
cd backend

# Check current migration version
uv run alembic current

# Create new migration
uv run alembic revision -m "descriptive_message"

# Apply migrations
uv run alembic upgrade head

# View migration history
uv run alembic history

# Downgrade one version
uv run alembic downgrade -1
```

## Critical Rules

### 🔴 Rule 1: NEVER Create Migrations Manually

**Always use the CLI to generate migration files**. Never create migration files from scratch.

```bash
# ✅ CORRECT
uv run alembic revision -m "add_user_preferences_table"

# ❌ WRONG
touch alembic/versions/abc123_add_user_preferences_table.py
```

**Why**: The CLI generates the proper file structure, revision IDs, and links to the migration chain.

### 🔴 Rule 2: Migrations Are IMMUTABLE

**NEVER edit an existing migration that may have been applied in any environment**.

Migrations must be treated as **immutable** once they leave your local machine.

**Exception**: You may edit a migration ONLY if:
- You are 100% certain it has never been applied anywhere (truly local-only)
- It has never been committed to version control
- No other developer has based work on it

**If you need to change behavior**:
- Write a **new forward-only corrective migration**
- Examples: revoke grants, fix policies, backfill data, add missing indexes

```bash
# ✅ CORRECT: Create corrective migration
uv run alembic revision -m "fix_user_table_rls_policy"

# ❌ WRONG: Edit existing migration
vim alembic/versions/abc123_add_user_table.py
```

### 🔴 Rule 3: Migrations Must Be Idempotent

Make DDL and security operations **idempotent** whenever practical.

Use safe SQL patterns:
- `CREATE ... IF NOT EXISTS`
- `DROP ... IF EXISTS`
- Safe `REVOKE`/`GRANT` patterns that don't fail if already applied

```python
# ✅ CORRECT: Idempotent
def upgrade() -> None:
    op.execute("CREATE SCHEMA IF NOT EXISTS sessions")
    op.execute("CREATE EXTENSION IF NOT EXISTS vector")

# ❌ WRONG: Will fail on re-run
def upgrade() -> None:
    op.execute("CREATE SCHEMA sessions")
    op.execute("CREATE EXTENSION vector")
```

### 🔴 Rule 4: Row Level Security (RLS) Is Mandatory

Whenever modifying schema (tables, columns, views), **MUST** verify and update RLS policies.

**Security Checklist for New Tables**:
1. Enable RLS: `ALTER TABLE ... ENABLE ROW LEVEL SECURITY`
2. Create policies for all operations: `SELECT`, `INSERT`, `UPDATE`, `DELETE`
3. For vector embeddings: Ensure strict RLS to prevent unauthorized semantic search

**For Existing Tables**:
- Review existing policies when adding columns
- Ensure policies still provide adequate protection
- Test that authorized users can access, unauthorized cannot

```python
# ✅ CORRECT: Enable RLS on new table
def upgrade() -> None:
    # Create table
    op.create_table(
        'user_preferences',
        sa.Column('id', sa.UUID(), nullable=False),
        sa.Column('user_id', sa.UUID(), nullable=False),
        schema='sessions'
    )

    # Enable RLS
    op.execute("ALTER TABLE sessions.user_preferences ENABLE ROW LEVEL SECURITY")

    # Create policy
    op.execute("""
        CREATE POLICY user_preferences_select ON sessions.user_preferences
        FOR SELECT USING (user_id = public.current_user_id())
    """)
```

### ⚠️ Rule 5: Avoid Autogenerate

**Do NOT use `--autogenerate`** as it is unreliable and creates false positives.

```bash
# ❌ AVOID
uv run alembic revision --autogenerate -m "message"

# ✅ USE
uv run alembic revision -m "message"
# Then manually write upgrade/downgrade functions
```

If you must use `--autogenerate`:
- Strictly review every line of generated code
- Remove false positives
- Add missing RLS policies
- Test thoroughly

### 🔐 Rule 6: No Secrets in Migrations

**Never embed secrets** (passwords, API keys) directly in migration code.

**Always use environment variables**:

```python
# ✅ CORRECT
import os
from alembic import op

def upgrade() -> None:
    db_user = os.getenv("DB_APP_USER", "app")
    db_password = os.getenv("DB_APP_PASSWORD")

    op.execute(
        f"CREATE USER {db_user} WITH PASSWORD '{db_password}'"
    )

# ❌ WRONG
def upgrade() -> None:
    op.execute("CREATE USER aicc WITH PASSWORD 'hardcoded_secret'")
```

## Creating a Migration

### Step 1: Generate Migration File

```bash
cd backend
uv run alembic revision -m "add_user_session_tracking"
```

This creates: `alembic/versions/<revision>_add_user_session_tracking.py`

### Step 2: Write Upgrade Function

Edit the generated file:

```python
"""add_user_session_tracking

Revision ID: abc123def456
Revises: previous_revision
Create Date: 2026-01-16 14:00:00

"""
from typing import Sequence, Union
from alembic import op
import sqlalchemy as sa

revision: str = 'abc123def456'
down_revision: Union[str, Sequence[str], None] = 'previous_revision'
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    """Upgrade schema."""
    # Create table
    op.create_table(
        'user_sessions',
        sa.Column('id', sa.UUID(), server_default=sa.text('gen_random_uuid()'), nullable=False),
        sa.Column('user_id', sa.UUID(), nullable=False),
        sa.Column('session_token', sa.String(), nullable=False),
        sa.Column('created_at', sa.DateTime(timezone=True), server_default=sa.text('now()'), nullable=False),
        sa.Column('expires_at', sa.DateTime(timezone=True), nullable=False),
        sa.PrimaryKeyConstraint('id'),
        schema='sessions'
    )

    # Create index
    op.create_index(
        'ix_user_sessions_user_id',
        'user_sessions',
        ['user_id'],
        schema='sessions'
    )

    # Enable RLS
    op.execute("ALTER TABLE sessions.user_sessions ENABLE ROW LEVEL SECURITY")

    # Create RLS policies
    op.execute("""
        CREATE POLICY user_sessions_select ON sessions.user_sessions
        FOR SELECT USING (user_id = public.current_user_id())
    """)

    op.execute("""
        CREATE POLICY user_sessions_insert ON sessions.user_sessions
        FOR INSERT WITH CHECK (user_id = public.current_user_id())
    """)


def downgrade() -> None:
    """Downgrade schema."""
    op.drop_table('user_sessions', schema='sessions')
```

### Step 3: Test Migration

```bash
# Apply migration
uv run alembic upgrade head

# Verify it worked
uv run alembic current

# Test downgrade (in local env only)
uv run alembic downgrade -1

# Re-apply
uv run alembic upgrade head
```

### Step 4: Verify RLS

Connect to database and verify:

```sql
-- Check RLS is enabled
SELECT tablename, rowsecurity
FROM pg_tables
WHERE schemaname = 'sessions' AND tablename = 'user_sessions';

-- Check policies exist
SELECT * FROM pg_policies WHERE tablename = 'user_sessions';

-- Test access control
SET app.user.id = 'valid-uuid';
SELECT * FROM sessions.user_sessions;  -- Should only return user's sessions
```

## Common Migration Patterns

### Pattern 1: Add Column

```python
def upgrade() -> None:
    op.add_column(
        'team_state',
        sa.Column('metadata', sa.JSON(), nullable=True),
        schema='sessions'
    )
```

### Pattern 2: Create Index

```python
def upgrade() -> None:
    op.create_index(
        'ix_file_metadata_session_id',
        'file_metadata',
        ['session_id'],
        schema='files',
        if_not_exists=True  # Idempotent!
    )
```

### Pattern 3: Add Foreign Key

```python
def upgrade() -> None:
    op.create_foreign_key(
        'fk_canvas_session',
        'canvas',
        'team_state',
        ['session_id'],
        ['session_id'],
        source_schema='sessions',
        referent_schema='sessions',
        ondelete='CASCADE'
    )
```

### Pattern 4: Create View with RLS

```python
def upgrade() -> None:
    # Create view
    op.execute("""
        CREATE OR REPLACE VIEW sessions.user_active_sessions AS
        SELECT id, user_id, created_at
        FROM sessions.user_sessions
        WHERE expires_at > now()
    """)

    # Note: Views inherit RLS from underlying tables by default (SECURITY INVOKER)
    # Explicitly set if needed:
    op.execute("""
        ALTER VIEW sessions.user_active_sessions
        SET (security_invoker = true)
    """)
```

### Pattern 5: Create Function (Helper for RLS)

```python
def upgrade() -> None:
    op.execute("""
        CREATE OR REPLACE FUNCTION public.current_user_id() RETURNS uuid AS $$
            SELECT nullif(current_setting('jwt.claims.sub', true), '')::uuid;
        $$ LANGUAGE sql STABLE;
    """)

def downgrade() -> None:
    op.execute("DROP FUNCTION IF EXISTS public.current_user_id()")
```

### Pattern 6: Grant/Revoke Permissions

```python
def upgrade() -> None:
    import os

    app_user = os.getenv("DB_APP_USER", "app")

    # Idempotent grants
    op.execute(f"GRANT USAGE ON SCHEMA sessions TO {app_user}")
    op.execute(f"GRANT ALL ON ALL TABLES IN SCHEMA sessions TO {app_user}")

    # Revoke from library schema
    op.execute(f"REVOKE ALL ON SCHEMA library FROM {app_user}")
```

## Migration Workflow Best Practices

### 1. Plan Your Migration

Before creating the migration:
- Understand the schema change needed
- Consider impact on existing data
- Plan RLS policies
- Think about rollback strategy

### 2. Use Descriptive Names

```bash
# ✅ GOOD
uv run alembic revision -m "add_canvas_sections_table_with_rls"
uv run alembic revision -m "create_library_mcp_readonly_user"

# ❌ BAD
uv run alembic revision -m "update"
uv run alembic revision -m "fix"
```

### 3. Keep Migrations Focused

One migration should do one logical thing:
- ✅ Add a table
- ✅ Add RLS policies to a table
- ✅ Create an index
- ❌ Add 5 tables + 10 indexes + change 3 columns

### 4. Write Reversible Downgrade Functions

Always implement `downgrade()` when possible:

```python
def upgrade() -> None:
    op.add_column('users', sa.Column('email', sa.String()))

def downgrade() -> None:
    op.drop_column('users', 'email')
```

### 5. Test in Local Environment First

```bash
# Apply
uv run alembic upgrade head

# Run application tests
uv run pytest

# Test downgrade
uv run alembic downgrade -1

# Re-apply
uv run alembic upgrade head
```

### 6. Document Complex Migrations

Add comments explaining non-obvious logic:

```python
def upgrade() -> None:
    """
    Add organization_id to support multi-tenant isolation.

    Note: Existing rows will have NULL organization_id.
    A follow-up data migration will populate these values.
    """
    op.add_column(
        'team_state',
        sa.Column('organization_id', sa.UUID(), nullable=True),
        schema='sessions'
    )
```

## Troubleshooting

### "Target database is not up to date"

**Cause**: Your local database is behind the current migration head.

**Solution**:
```bash
uv run alembic upgrade head
```

### "Multiple head revisions are present"

**Cause**: Migration history has branched (see alembic-migration-rebase skill).

**Solution**:
```bash
# Merge heads
uv run alembic merge -m "merge heads" head1 head2

# Or use the alembic-migration-rebase skill
```

### "Can't locate revision"

**Cause**: Database has a migration that doesn't exist in your files.

**Solution**: Either restore the migration file or reset the database.

### Migration Fails Midway

**Cause**: Non-idempotent operations or SQL errors.

**Solution**:
```bash
# Check current state
uv run alembic current

# Fix the migration file (add IF NOT EXISTS, etc.)
# If migration partially applied, may need to manually clean up

# Try again
uv run alembic upgrade head
```

## RLS-Specific Patterns

### Enable RLS on New Table

```python
def upgrade() -> None:
    # Create table first
    op.create_table('sensitive_data', ...)

    # Enable RLS
    op.execute("ALTER TABLE sessions.sensitive_data ENABLE ROW LEVEL SECURITY")
```

### Create Comprehensive RLS Policies

```python
def upgrade() -> None:
    # SELECT policy
    op.execute("""
        CREATE POLICY sensitive_data_select ON sessions.sensitive_data
        FOR SELECT USING (user_id = public.current_user_id())
    """)

    # INSERT policy
    op.execute("""
        CREATE POLICY sensitive_data_insert ON sessions.sensitive_data
        FOR INSERT WITH CHECK (user_id = public.current_user_id())
    """)

    # UPDATE policy
    op.execute("""
        CREATE POLICY sensitive_data_update ON sessions.sensitive_data
        FOR UPDATE USING (user_id = public.current_user_id())
        WITH CHECK (user_id = public.current_user_id())
    """)

    # DELETE policy
    op.execute("""
        CREATE POLICY sensitive_data_delete ON sessions.sensitive_data
        FOR DELETE USING (user_id = public.current_user_id())
    """)
```

### Vector Embedding Tables RLS

**Critical**: Vector embeddings must have strict RLS to prevent unauthorized semantic search.

```python
def upgrade() -> None:
    # Enable RLS on embedding table
    op.execute("ALTER TABLE files.file_metadata_embedding ENABLE ROW LEVEL SECURITY")

    # Policy should join back to parent table to inherit access rules
    op.execute("""
        CREATE POLICY file_embedding_select ON files.file_metadata_embedding
        FOR SELECT USING (
            EXISTS (
                SELECT 1 FROM files.file_metadata fm
                WHERE fm.id = file_metadata_embedding.file_id
                AND fm.user_id = public.current_user_id()
            )
        )
    """)
```

## Integration with Project Workflow

### With Docker Compose

Migrations run automatically via the `migrations` service:

```bash
# Start stack (migrations run automatically)
docker-compose up -d postgres migrations vectorizer-worker

# Check migration logs
docker logs migrations
```

### Manual Migration Run

```bash
# From repo root
cd backend
uv run alembic upgrade head
```

## Related Resources

- Skill: `alembic-migration-rebase` (for rebasing migrations)
- [Alembic Documentation](https://alembic.sqlalchemy.org/)
- [PostgreSQL RLS Documentation](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)

## Checklist for New Migrations

Before committing a migration:

- [ ] Used CLI to generate migration file (`uv run alembic revision -m "..."`)
- [ ] Wrote clear, focused upgrade function
- [ ] Implemented downgrade function (when possible)
- [ ] Made operations idempotent (IF NOT EXISTS, IF EXISTS)
- [ ] Enabled RLS on any new tables
- [ ] Created policies for SELECT, INSERT, UPDATE, DELETE
- [ ] No hardcoded secrets (use environment variables)
- [ ] Tested upgrade in local environment
- [ ] Tested downgrade (if reversible)
- [ ] Verified RLS policies work correctly
- [ ] Ran application tests
- [ ] Used descriptive migration message
- [ ] Added comments for complex logic

## Quick Command Reference

```bash
# Navigate to alembic directory
cd backend

# Create migration
uv run alembic revision -m "description"

# Check current version
uv run alembic current

# View history
uv run alembic history

# Apply all pending migrations
uv run alembic upgrade head

# Upgrade to specific version
uv run alembic upgrade abc123

# Downgrade one version
uv run alembic downgrade -1

# Downgrade to specific version
uv run alembic downgrade abc123

# Show heads (in branched history)
uv run alembic heads

# Show branches
uv run alembic branches

# Merge heads
uv run alembic merge -m "merge" head1 head2
```
