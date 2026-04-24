---
name: postgresql-rls
description: Implement PostgreSQL Row Level Security (RLS) with FastAPI, SQLAlchemy (Async), and Alembic. Covers JWT-based identity injection, helper functions, policy patterns, view/vector search security, connection pooling safety, and integration testing. Use when adding user-level data isolation to any FastAPI + PostgreSQL application.
---

# PostgreSQL Row Level Security (RLS) with FastAPI & SQLAlchemy

A comprehensive guide for implementing **defense-in-depth authorization** using PostgreSQL RLS in a Python web application stack (FastAPI + SQLAlchemy Async + Alembic).

---

## 1. Core Concept

RLS moves authorization enforcement **from the application layer into the database layer**. Even if the API has a bug, the database itself prevents unauthorized data access.

**The pattern:**

1. **API Layer** — Validates JWT, extracts user identity (`sub` claim).
2. **Database Session** — Injects identity into the PostgreSQL connection via `set_config`.
3. **PostgreSQL** — RLS policies automatically filter every query to only return rows the user owns.

```
┌──────────┐      ┌──────────────────┐      ┌─────────────────────┐
│  Client   │─JWT─▶│  FastAPI + Auth  │──────▶│  PostgreSQL + RLS   │
│           │      │  set_config(sub) │      │  WHERE user_id =    │
│           │◀─────│  yield session   │◀─────│  current_user_id()  │
└──────────┘      └──────────────────┘      └─────────────────────┘
```

---

## 2. PostgreSQL Helper Functions

These helper functions live in the `public` schema and are referenced by all RLS policies. **Create them in an Alembic migration** before any policy that depends on them.

### 2.1 `current_user_id()` — The Core Identity Function

Returns the UUID of the currently authenticated user. This is the single source of truth for all RLS policies.

```sql
CREATE OR REPLACE FUNCTION public.current_user_id() RETURNS uuid AS $$
    SELECT nullif(current_setting('jwt.claims.sub', true), '')::uuid;
$$ LANGUAGE sql STABLE;
```

**Key details:**
- `current_setting('jwt.claims.sub', true)` — The `true` flag means "return NULL instead of error if the setting doesn't exist". This prevents crashes on uninitialized connections.
- `nullif(..., '')` — Converts empty strings to NULL, ensuring no accidental match against rows with NULL `user_id`.
- `STABLE` — Tells the query planner this function returns the same value within a single statement, enabling optimization.
- Returns `uuid` — All `user_id` columns must be UUID type.

### 2.2 `current_user_role()` — Optional Role Function

If your application supports role-based differentiation (e.g., admin bypass, service accounts):

```sql
CREATE OR REPLACE FUNCTION public.current_user_role() RETURNS text AS $$
    SELECT nullif(current_setting('jwt.claims.role', true), '');
$$ LANGUAGE sql STABLE;
```

### 2.3 Migration Example

```python
"""create_rls_helper_functions

Revision ID: <generated>
Revises: <previous>
"""
from alembic import op


def upgrade() -> None:
    # Core identity function — referenced by all RLS policies
    op.execute("""
        CREATE OR REPLACE FUNCTION public.current_user_id() RETURNS uuid AS $$
            SELECT nullif(current_setting('jwt.claims.sub', true), '')::uuid;
        $$ LANGUAGE sql STABLE;
    """)

    # Optional: role function
    op.execute("""
        CREATE OR REPLACE FUNCTION public.current_user_role() RETURNS text AS $$
            SELECT nullif(current_setting('jwt.claims.role', true), '');
        $$ LANGUAGE sql STABLE;
    """)


def downgrade() -> None:
    op.execute("DROP FUNCTION IF EXISTS public.current_user_role()")
    op.execute("DROP FUNCTION IF EXISTS public.current_user_id()")
```

---

## 3. RLS Policy Patterns

### 3.1 Pattern: Direct Ownership

Use when the table has a direct `user_id` column.

```sql
-- Enable RLS on the table
ALTER TABLE my_schema.projects ENABLE ROW LEVEL SECURITY;

-- Single policy covering all operations
CREATE POLICY project_isolation_policy ON my_schema.projects
    FOR ALL
    USING (user_id = public.current_user_id())
    WITH CHECK (user_id = public.current_user_id());
```

**Explanation:**
- `USING` — Filters rows on `SELECT`, `UPDATE`, `DELETE`. Only rows where `user_id` matches are visible.
- `WITH CHECK` — Validates rows on `INSERT` and `UPDATE`. Prevents writing rows with a different `user_id`.

### 3.2 Pattern: Relationship-Based Access (Child Tables)

Use when a table inherits permissions from a parent (e.g., files belong to projects, projects have `user_id`).

```sql
ALTER TABLE my_schema.files ENABLE ROW LEVEL SECURITY;

CREATE POLICY file_isolation_policy ON my_schema.files
    FOR ALL
    USING (
        EXISTS (
            SELECT 1
            FROM my_schema.projects p
            WHERE p.id = my_schema.files.project_id
              AND p.user_id = public.current_user_id()
        )
    )
    WITH CHECK (
        EXISTS (
            SELECT 1
            FROM my_schema.projects p
            WHERE p.id = my_schema.files.project_id
              AND p.user_id = public.current_user_id()
        )
    );
```

**When to use:**
- The table has no direct `user_id` column.
- Ownership is determined through a foreign key chain (e.g., `file → project → user`).

### 3.3 Pattern: Vector Embedding Inheritance

Critical for securing vector/semantic search. The embedding table inherits access rules from its parent metadata table.

```sql
ALTER TABLE my_schema.embeddings ENABLE ROW LEVEL SECURITY;

CREATE POLICY embedding_isolation_policy ON my_schema.embeddings
    FOR ALL
    USING (
        EXISTS (
            SELECT 1
            FROM my_schema.files f
            WHERE f.id = my_schema.embeddings.file_id
            -- The RLS policy on my_schema.files already filters by user,
            -- so this EXISTS naturally inherits that restriction.
        )
    );
```

**Why this matters:**
- Semantic search queries (`ORDER BY embedding <=> query_vector`) scan broadly.
- Without this policy, a user could retrieve embeddings (and infer content) of other users' files.
- The `EXISTS` subquery against the parent table **automatically** inherits the parent's RLS policy — PostgreSQL applies policies recursively.

### 3.4 Pattern: Split Policies Per Operation

For finer-grained control, create separate policies per DML operation:

```sql
-- Read: user can see their own rows
CREATE POLICY select_own ON my_schema.items
    FOR SELECT
    USING (user_id = public.current_user_id());

-- Insert: user can only insert rows they own
CREATE POLICY insert_own ON my_schema.items
    FOR INSERT
    WITH CHECK (user_id = public.current_user_id());

-- Update: user can only modify their own rows
CREATE POLICY update_own ON my_schema.items
    FOR UPDATE
    USING (user_id = public.current_user_id())
    WITH CHECK (user_id = public.current_user_id());

-- Delete: user can only delete their own rows
CREATE POLICY delete_own ON my_schema.items
    FOR DELETE
    USING (user_id = public.current_user_id());
```

### 3.5 Pattern: Admin/Service Bypass

If you need service accounts or admins to bypass RLS:

```sql
CREATE POLICY admin_bypass ON my_schema.items
    FOR ALL
    USING (public.current_user_role() = 'service')
    WITH CHECK (public.current_user_role() = 'service');
```

> **Warning**: Prefer granting `BYPASSRLS` to a dedicated superuser role over policy-based bypass. Policy-based bypass adds complexity and attack surface.

---

## 4. Views & Security Invoker

Database views are a common way to join data across tables. When RLS is active, view security mode matters.

### 4.1 `SECURITY INVOKER` (Default — Use This)

The view executes with the **caller's** permissions. RLS policies on underlying tables are applied automatically.

```sql
CREATE OR REPLACE VIEW my_schema.active_user_items AS
SELECT i.id, i.name, i.created_at
FROM my_schema.items i
WHERE i.is_active = true;

-- Explicitly set if needed (PostgreSQL 15+):
ALTER VIEW my_schema.active_user_items SET (security_invoker = true);
```

**Result:** When a user queries this view, they only see their own active items — the RLS policy on `my_schema.items` filters automatically.

### 4.2 `SECURITY DEFINER` (Avoid for RLS)

The view runs as the **view owner** (typically a superuser or privileged role). This **bypasses RLS** and should NOT be used for user-facing views.

```sql
-- ❌ AVOID: This bypasses RLS
CREATE VIEW my_schema.all_items_admin
WITH (security_invoker = false)
AS SELECT * FROM my_schema.items;
```

### 4.3 Views Joining Multiple RLS-Protected Tables

When a view joins tables that each have their own RLS policies, PostgreSQL applies **all** policies independently:

```sql
CREATE OR REPLACE VIEW my_schema.file_details AS
SELECT
    f.id AS file_id,
    f.name AS file_name,
    p.name AS project_name
FROM my_schema.files f
JOIN my_schema.projects p ON p.id = f.project_id;
```

Both `files` and `projects` RLS policies are enforced. The user only sees rows where they own the project **and** have access to the file.

---

## 5. FastAPI Integration

### 5.1 JWT Authentication Dependency

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer
import jwt
from pydantic import BaseModel
from uuid import UUID

security = HTTPBearer()


class AuthUser(BaseModel):
    id: UUID
    role: str | None = None


async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
) -> AuthUser:
    """Decode and validate JWT, return the authenticated user."""
    token = credentials.credentials
    try:
        payload = jwt.decode(
            token,
            key=settings.JWT_SIGNING_KEY,
            algorithms=[settings.JWT_ALGORITHM],
            audience=settings.JWT_AUDIENCE,
        )
    except jwt.InvalidTokenError as e:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid or expired token",
        ) from e

    sub = payload.get("sub")
    if not sub:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token missing 'sub' claim",
        )

    return AuthUser(id=UUID(sub), role=payload.get("role"))
```

### 5.2 Authenticated Database Session (The Transaction Wrapper)

This is the **critical integration point**. It injects the user identity into the PostgreSQL session at the start of every transaction.

```python
from collections.abc import AsyncGenerator

from fastapi import Depends
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker

from .auth_dependencies import AuthUser, get_current_user


async def get_authenticated_db(
    user: AuthUser = Depends(get_current_user),
    session_factory: async_sessionmaker[AsyncSession] = Depends(get_session_factory),
) -> AsyncGenerator[AsyncSession, None]:
    """
    Provide a database session with RLS context.

    Injects the authenticated user's UUID into the PostgreSQL
    transaction-local variable `jwt.claims.sub`. All subsequent
    queries in this session are automatically filtered by RLS policies.
    """
    async with session_factory() as session:
        async with session.begin():
            # CRITICAL: is_local=true (3rd param) scopes the setting
            # to this transaction only. When the connection returns to
            # the pool, the setting is automatically discarded.
            await session.execute(
                text("SELECT set_config('jwt.claims.sub', :sub, true)"),
                {"sub": str(user.id)},
            )
            yield session
```

**Why `is_local=true` matters:**
- Without it, the setting persists on the connection after the transaction ends.
- When the connection returns to the pool and is reused by another user, they would inherit the previous user's identity.
- `true` ensures the setting is automatically cleared on `COMMIT` or `ROLLBACK`.

### 5.3 Using the Dependency in Routes

```python
from fastapi import APIRouter, Depends
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

router = APIRouter()


@router.get("/projects")
async def list_projects(
    db: AsyncSession = Depends(get_authenticated_db),
):
    """List all projects for the authenticated user.
    
    No WHERE clause needed — RLS handles filtering automatically.
    """
    result = await db.execute(select(Project))
    return result.scalars().all()


@router.get("/projects/{project_id}")
async def get_project(
    project_id: UUID,
    db: AsyncSession = Depends(get_authenticated_db),
):
    """Get a single project. Returns 404 if not found OR not owned by user."""
    result = await db.execute(
        select(Project).where(Project.id == project_id)
    )
    project = result.scalar_one_or_none()
    if not project:
        raise HTTPException(status_code=404, detail="Project not found")
    return project
```

**Key insight:** The route code never mentions `user_id`. The database filters transparently — queries look identical to unprotected queries but return only authorized data.

---

## 6. Connection Pooling Safety

### 6.1 The Risk

With connection pooling (asyncpg, psycopg pool, pgbouncer), the same physical connection serves many users. If `set_config` persists beyond a transaction, the next user on that connection inherits the previous user's identity.

### 6.2 The Mitigation

1. **Always use `is_local=true`** in `set_config` (scopes to current transaction).
2. **Always wrap the session in an explicit transaction** (`session.begin()`).
3. **Never** use `set_config` with `false` (session-level persistence).

### 6.3 Verification Query

Run this after a transaction ends to confirm the setting is cleared:

```sql
-- After COMMIT/ROLLBACK, this should return '' or NULL
SELECT current_setting('jwt.claims.sub', true);
```

---

## 7. Database Role Configuration

### 7.1 Application User (Non-Superuser)

The application must connect as a **non-superuser** role. Superusers bypass all RLS policies.

```sql
-- Create the application role
CREATE ROLE app_user WITH LOGIN PASSWORD :'password' NOSUPERUSER NOBYPASSRLS;

-- Grant schema access
GRANT USAGE ON SCHEMA my_schema TO app_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA my_schema TO app_user;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA my_schema TO app_user;

-- Set default privileges for future tables
ALTER DEFAULT PRIVILEGES IN SCHEMA my_schema
    GRANT ALL PRIVILEGES ON TABLES TO app_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA my_schema
    GRANT ALL PRIVILEGES ON SEQUENCES TO app_user;
```

### 7.2 Migration User

Migrations typically need broader privileges. Use a separate role or the database owner:

```sql
-- Migrations run as the DB owner (can create/alter policies)
-- Application runs as app_user (subject to RLS)
```

### 7.3 Verification

```sql
-- Confirm the app user does NOT bypass RLS
SELECT rolname, rolsuper, rolbypassrls
FROM pg_roles
WHERE rolname = 'app_user';
-- Expected: rolsuper=false, rolbypassrls=false
```

---

## 8. Alembic Migration Patterns for RLS

### 8.1 Enable RLS + Create Policy

```python
def upgrade() -> None:
    # Enable RLS
    op.execute("ALTER TABLE my_schema.items ENABLE ROW LEVEL SECURITY")

    # Create policy (idempotent via DROP IF EXISTS + CREATE)
    op.execute("DROP POLICY IF EXISTS items_isolation ON my_schema.items")
    op.execute("""
        CREATE POLICY items_isolation ON my_schema.items
            FOR ALL
            USING (user_id = public.current_user_id())
            WITH CHECK (user_id = public.current_user_id())
    """)


def downgrade() -> None:
    op.execute("DROP POLICY IF EXISTS items_isolation ON my_schema.items")
    op.execute("ALTER TABLE my_schema.items DISABLE ROW LEVEL SECURITY")
```

### 8.2 Add RLS to an Existing Table (Backfill User IDs First)

```python
def upgrade() -> None:
    # Step 1: Add user_id column if missing
    op.add_column(
        'legacy_items',
        sa.Column('user_id', sa.UUID(), nullable=True),
        schema='my_schema',
    )

    # Step 2: Backfill user_id from parent relationship
    op.execute("""
        UPDATE my_schema.legacy_items li
        SET user_id = p.user_id
        FROM my_schema.projects p
        WHERE p.id = li.project_id
    """)

    # Step 3: Make non-nullable after backfill
    op.alter_column(
        'legacy_items', 'user_id',
        nullable=False,
        schema='my_schema',
    )

    # Step 4: Enable RLS
    op.execute("ALTER TABLE my_schema.legacy_items ENABLE ROW LEVEL SECURITY")
    op.execute("""
        CREATE POLICY legacy_items_isolation ON my_schema.legacy_items
            FOR ALL
            USING (user_id = public.current_user_id())
            WITH CHECK (user_id = public.current_user_id())
    """)
```

### 8.3 Policy on Embedding/Vector Tables

```python
def upgrade() -> None:
    op.execute(
        "ALTER TABLE my_schema.document_embeddings ENABLE ROW LEVEL SECURITY"
    )
    op.execute("""
        CREATE POLICY embedding_isolation ON my_schema.document_embeddings
            FOR ALL
            USING (
                EXISTS (
                    SELECT 1
                    FROM my_schema.documents d
                    WHERE d.id = my_schema.document_embeddings.document_id
                )
            )
    """)
```

> The `EXISTS` subquery against `documents` is **itself** filtered by the RLS policy on `documents`. This creates a transitive chain: embeddings → documents → user.

---

## 9. Testing Strategy

### 9.1 Test Fixture: Authenticated Session

```python
import pytest
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession
from uuid import uuid4


ALICE_ID = uuid4()
BOB_ID = uuid4()


async def set_user_context(session: AsyncSession, user_id) -> None:
    """Simulate authenticated user in the DB session."""
    await session.execute(
        text("SELECT set_config('jwt.claims.sub', :sub, true)"),
        {"sub": str(user_id)},
    )


@pytest.fixture
async def alice_session(db_session: AsyncSession):
    """Database session authenticated as Alice."""
    await set_user_context(db_session, ALICE_ID)
    yield db_session


@pytest.fixture
async def bob_session(db_session: AsyncSession):
    """Database session authenticated as Bob."""
    await set_user_context(db_session, BOB_ID)
    yield db_session
```

### 9.2 Test: Resource Isolation

```python
async def test_user_cannot_see_other_users_projects(
    alice_session: AsyncSession,
    bob_session: AsyncSession,
):
    """Alice's projects are invisible to Bob."""
    # Alice creates a project
    await alice_session.execute(
        text("""
            INSERT INTO my_schema.projects (id, user_id, name)
            VALUES (:id, :user_id, :name)
        """),
        {"id": str(uuid4()), "user_id": str(ALICE_ID), "name": "Alice's Project"},
    )
    await alice_session.commit()

    # Bob queries projects
    result = await bob_session.execute(text("SELECT * FROM my_schema.projects"))
    rows = result.fetchall()

    assert len(rows) == 0, "Bob should not see Alice's projects"
```

### 9.3 Test: Connection Pool Leakage

```python
async def test_no_identity_leakage_between_requests(
    session_factory,
):
    """Sequential sessions on the same connection must not leak identity."""
    # Request 1: Alice
    async with session_factory() as session:
        async with session.begin():
            await set_user_context(session, ALICE_ID)
            await session.execute(
                text("""
                    INSERT INTO my_schema.projects (id, user_id, name)
                    VALUES (:id, :user_id, 'Alice Project')
                """),
                {"id": str(uuid4()), "user_id": str(ALICE_ID)},
            )

    # Request 2: Bob (same pool, possibly same connection)
    async with session_factory() as session:
        async with session.begin():
            await set_user_context(session, BOB_ID)
            result = await session.execute(
                text("SELECT * FROM my_schema.projects")
            )
            rows = result.fetchall()

    assert len(rows) == 0, "Bob must not see Alice's project from previous connection"
```

### 9.4 Test: Vector Search Isolation

```python
async def test_vector_search_respects_rls(
    alice_session: AsyncSession,
    bob_session: AsyncSession,
):
    """Semantic search must not return embeddings from other users' documents."""
    # Alice uploads a document + embedding
    doc_id = uuid4()
    await alice_session.execute(
        text("""
            INSERT INTO my_schema.documents (id, user_id, content)
            VALUES (:id, :user_id, 'secret content')
        """),
        {"id": str(doc_id), "user_id": str(ALICE_ID)},
    )
    await alice_session.execute(
        text("""
            INSERT INTO my_schema.document_embeddings (document_id, embedding)
            VALUES (:doc_id, :embedding)
        """),
        {"doc_id": str(doc_id), "embedding": "[0.1,0.2,0.3]"},  # pgvector format
    )
    await alice_session.commit()

    # Bob performs semantic search
    result = await bob_session.execute(
        text("""
            SELECT document_id
            FROM my_schema.document_embeddings
            ORDER BY embedding <=> ':query'::vector
            LIMIT 10
        """),
        {"query": "[0.1,0.2,0.3]"},
    )
    rows = result.fetchall()

    assert len(rows) == 0, "Bob must not find Alice's embeddings via vector search"
```

### 9.5 Test: View Inherits RLS

```python
async def test_view_applies_underlying_rls(
    alice_session: AsyncSession,
    bob_session: AsyncSession,
):
    """Views using SECURITY INVOKER must respect RLS on base tables."""
    # Seed data as Alice
    # ...

    # Bob queries the view
    result = await bob_session.execute(
        text("SELECT * FROM my_schema.active_user_items")
    )
    rows = result.fetchall()

    assert len(rows) == 0, "View must filter via underlying table's RLS"
```

---

## 10. Security Checklist

Use this checklist when implementing or reviewing RLS:

- [ ] **Helper function exists**: `public.current_user_id()` is deployed.
- [ ] **RLS enabled**: `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` on every table with sensitive data.
- [ ] **Policies created**: Every RLS-enabled table has policies covering all needed operations (`SELECT`, `INSERT`, `UPDATE`, `DELETE` or `ALL`).
- [ ] **Child tables covered**: Tables without direct `user_id` use `EXISTS` joins to inherit access from parent tables.
- [ ] **Vector/embedding tables secured**: Embedding tables have policies that chain back to the owning entity.
- [ ] **Views use SECURITY INVOKER**: No `SECURITY DEFINER` views on RLS-protected tables (unless intentional bypass).
- [ ] **App connects as non-superuser**: The application database role has `NOSUPERUSER NOBYPASSRLS`.
- [ ] **`set_config` uses `is_local=true`**: Every call to `set_config` passes `true` as the third argument.
- [ ] **Parameterized queries**: User identity is injected via `:sub` parameter, never string interpolation.
- [ ] **Tests pass**: Isolation tests, leakage tests, and vector search tests all green.
- [ ] **No `FORCE ROW LEVEL SECURITY`**: Unless you explicitly want RLS to apply to table owners too. (Default: table owner bypasses own RLS.)

---

## 11. Troubleshooting

### "Query returns all rows, RLS is not filtering"

1. **Check the connection role**: `SELECT current_user;` — if it's a superuser or the table owner, RLS is bypassed.
2. **Check RLS is enabled**: `SELECT rowsecurity FROM pg_tables WHERE tablename = 'your_table';` — must be `true`.
3. **Check `set_config` was called**: `SELECT current_setting('jwt.claims.sub', true);` — must return the expected UUID.
4. **Force RLS on table owner** (if the app role owns the table):
   ```sql
   ALTER TABLE my_schema.items FORCE ROW LEVEL SECURITY;
   ```

### "Query returns zero rows when it shouldn't"

1. **Check the user_id value**: Ensure the JWT `sub` matches the `user_id` stored in the row (same UUID format, no case mismatch).
2. **Check the helper function**: `SELECT public.current_user_id();` — should return a valid UUID, not NULL.
3. **Check for NULL user_id**: Rows with `NULL` in `user_id` will never match `= current_user_id()`.

### "Connection pool leaking identity"

1. **Verify `is_local=true`** in every `set_config` call.
2. **Verify transactions are properly committed/rolled back** before the connection returns to the pool.
3. **Test explicitly** with the leakage test pattern (Section 9.3).

### "RLS policy blocks migrations"

Migrations should run as the database owner or a superuser, which bypasses RLS. If using the app role for migrations, grant `BYPASSRLS` temporarily or use a separate migration role.
