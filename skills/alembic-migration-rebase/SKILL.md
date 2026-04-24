---
name: alembic-migration-rebase
description: Rebase and flatten Alembic database migrations that were created locally but never applied to production environments. Resolves branched migration history by merging heads and recreating migrations in a linear sequence. Use when you have multiple migration heads or migrations that need to be reorganized before production deployment.
---

# Alembic Migration Rebase Skill

This skill helps you rebase and flatten Alembic database migrations that exist only in your local environment. It's particularly useful when you have branched migration history or migrations that were created during development but never deployed to production.

## ⚠️ Critical Warnings

**ONLY use this skill for migrations that have NEVER been applied to production or shared environments!**

- ✅ **Safe to rebase**: Local-only migrations, development branch migrations, never-deployed migrations
- ❌ **NEVER rebase**: Migrations already applied in staging, production, or any shared environment
- ❌ **NEVER edit**: Existing migrations that may have been applied anywhere (always create new forward migrations instead)

**Why this matters**: Once a migration is applied in any non-local environment, it becomes part of the permanent history. Rebasing would create mismatches between environments and potentially cause data loss or corruption.

## Benefits of Linear Migration History

Maintaining a flat, linear migration history provides:

1. **Simplified debugging**: Easy to trace the sequence of schema changes
2. **Predictable rollbacks**: Clear upgrade/downgrade paths without merge complications
3. **Cleaner version control**: No merge conflicts in migration files
4. **Easier deployment**: No ambiguity about which migrations to apply
5. **Better collaboration**: Team members don't create conflicting migration branches

## When to Use This Skill

Use this skill when:

- **After rebasing your git branch** with another branch that contains migrations (to preserve linear history)
- You have multiple Alembic heads (branched migrations)
- Migrations were created on different git branches and now need to be unified
- You want to reorganize local-only migrations before pushing to production
- You see errors like "Multiple head revisions are present for given argument 'head'"
- You need to delete and recreate migrations that haven't been deployed

**Typical workflow**:
1. First, rebase your git branch (e.g., `git rebase main` or `git rebase feature-branch`)
2. Then, use this skill to rebase/flatten the Alembic migrations that resulted from the git rebase
3. This ensures both your git history AND migration history remain linear

## Prerequisites

Before starting:

1. Ensure migrations have NOT been applied to any non-local environment
2. Have a backup of your database (or be able to recreate it)
3. Know which migrations are the "real" base (from production/main branch)
4. Have uncommitted or stashed git changes saved

## Step-by-Step Process

### 1. Identify Migration Status

```bash
cd backend  # or your alembic location
uv run alembic current              # Check current database state
uv run alembic heads                # Check for multiple heads
uv run alembic branches             # See branch points
```

### 2. Find the Branch Point

Identify the last "real" migration (the one that exists in your main branch):

```bash
uv run alembic history              # Review full history
```

Look for the revision where branching occurred (marked with "branchpoint" in output).

### 3. Reset Database to Branch Point

Drop and recreate your database, or downgrade to the branch point:

```bash
# Option A: Fresh database
docker-compose down
docker volume rm <project>_db_data
docker-compose up -d postgres migrations

# Option B: Downgrade (if migrations support it)
uv run alembic downgrade <branch_point_revision>
```

### 4. Merge Any Heads

If you have multiple independent branches that both need to be kept:

```bash
uv run alembic merge -m "merge migration heads" <head1> <head2>
uv run alembic upgrade head
```

### 5. Delete Local-Only Migration Files

Remove the migration files that were never deployed:

```bash
# From the alembic/versions directory
rm backend/alembic/versions/<revision>_*.py
```

**Important**: Only delete migrations that:
- Were created locally
- Have never been applied to shared environments
- Are not referenced in production

### 6. Recreate Migrations

Create new migrations with the same content but in linear sequence:

```bash
# Create new migration files
uv run alembic revision -m "descriptive_migration_name"
```

Then edit the generated migration file to add the proper upgrade/downgrade logic.

### 7. Apply New Migrations

```bash
uv run alembic upgrade head
uv run alembic current              # Verify you're at the new head
```

### 8. Verify Database State

Test that your database schema is correct:

```bash
# Run application tests
uv run pytest

# Verify specific tables/columns exist
# Connect to DB and check schema manually if needed
```

### 9. Commit the Changes

```bash
git add backend/alembic/versions/
git commit -m "refactor: rebase and flatten alembic migrations"
```

## Common Patterns

### Pattern 1: Simple Branch Merge

When two branches created migrations from the same point:

```bash
# Identify heads
uv run alembic heads
# Output: abc123 (head), def456 (head)

# Merge them
uv run alembic merge -m "merge heads" abc123 def456
uv run alembic upgrade head
```

### Pattern 2: Delete and Recreate

When you want to completely restart from a known good state:

```bash
# 1. Note which migrations to recreate and their content
# 2. Reset database to branch point
# 3. Delete local migration files
# 4. Create new migrations in desired order
uv run alembic revision -m "first_migration"
uv run alembic revision -m "second_migration"
# 5. Fill in upgrade/downgrade functions
# 6. Apply: uv run alembic upgrade head
```

### Pattern 3: Preserve One Branch, Delete Another

When one branch has the "correct" changes:

```bash
# Keep branch A, delete branch B
# 1. Note the head of branch A
# 2. Downgrade to branch point
# 3. Delete branch B migration files
# 4. Upgrade to branch A head
uv run alembic upgrade <branch_a_head>
```

## Troubleshooting

### "Can't locate revision identified by '<revision>'"

**Cause**: Database has a migration applied that doesn't exist in your files.

**Solution**: Either restore the missing migration file, or reset the database:

```bash
# Reset alembic_version table
# In psql or database tool:
DELETE FROM alembic_version;

# Or recreate database from scratch
docker-compose down
docker volume rm <project>_db_data
docker-compose up -d postgres
```

### "Multiple head revisions are present"

**Cause**: Your migration history has branched.

**Solution**: Use the merge or delete-recreate patterns above.

### Migration Applied in Production Accidentally

**Cause**: A local-only migration was deployed.

**Solution**:
1. Do NOT delete or modify the migration
2. Create a new forward migration to correct any issues
3. Treat this migration as permanent going forward

## Best Practices

1. **Always verify migration status** before rebasing
2. **Keep a backup** of both database and migration files before major changes
3. **Test the rebase** in a local environment before committing
4. **Document the changes** in your commit message
5. **Coordinate with team** to avoid conflicts when rebasing shared branches
6. **Use feature branches** for experimental migrations that might need rebasing
7. **Mark migrations** with comments if they're experimental or not yet finalized

## Example Scenario

Here's a real example from our codebase:

```bash
# Context: Rebased git branch 25081/rls onto contracting-companion branch
# This brought in conflicting migrations from both branches
# Problem: Had 3 RLS migrations (670694995377, 18fba0cc047f, 0181c9b197a0)
# that branched from a1b2c3d4e5f6 but were never deployed
# Solution: Flatten the migrations to preserve linear history

# Step 1: Found branch point
uv run alembic history
# Output showed: a1b2c3d4e5f6 (branchpoint)

# Step 2: Reset database
docker-compose down && docker volume rm <project>_db_data
docker-compose up -d postgres migrations

# Step 3: Merged canvas changes branch
uv run alembic merge -m "merge canvas and rls branches" heads
uv run alembic upgrade head

# Step 4: Deleted old migrations
rm backend/alembic/versions/670694995377_*.py
rm backend/alembic/versions/18fba0cc047f_*.py
rm backend/alembic/versions/0181c9b197a0_*.py

# Step 5: Created new migrations in linear order
uv run alembic revision -m "add_current_user_id_helper_functions"
uv run alembic revision -m "harden_app_user_remove_library_access"
uv run alembic revision -m "create_library_mcp_db_user"

# Step 6: Filled in migration content and applied
uv run alembic upgrade head

# Result: Clean linear history from a1b2c3d4e5f6 → merge → RLS migrations
```

## Related Resources

- [Alembic Documentation - Branches](https://alembic.sqlalchemy.org/en/latest/branches.html)
- [Alembic Documentation - Cookbook](https://alembic.sqlalchemy.org/en/latest/cookbook.html)

## Safety Checklist

Before rebasing migrations, verify:

- [ ] These migrations have NEVER been applied to staging or production
- [ ] You have a database backup or can recreate it easily
- [ ] You've communicated with your team about the rebase
- [ ] You've documented which migrations you're removing and why
- [ ] You understand the content of each migration being recreated
- [ ] You've tested the new migrations in a local environment
- [ ] Your application still works after the rebase
- [ ] You've verified the database schema is correct

## When NOT to Use This Skill

Do NOT use this skill if:

- Migrations have been applied to any shared environment
- You're unsure if migrations have been deployed
- Other developers may have based work on these migrations
- The migrations have been merged to main/master branch
- More than a few days have passed since creating the migrations
- You're working in a production environment

Instead, create forward-only corrective migrations that work with the existing history.
