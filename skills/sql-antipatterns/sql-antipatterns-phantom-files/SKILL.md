---
name: sql-antipatterns-phantom-files

description: Detects the Phantom Files antipattern — storing images or other binary media as external files with only a pathname column in the database — which loses transactional guarantees (COMMIT/ROLLBACK), garbage collection, backup coverage, and access control, and guides toward a BLOB column when those guarantees matter.
---

# SQL Antipatterns — Phantom Files

This skill helps developers reason clearly about a genuinely two-sided
decision — store binary media as files referenced by path, or as `BLOB`
columns in the database — instead of following a blanket "files always
belong outside the database" rule that quietly drops transactional
guarantees.

---

## 1. Recognize the Antipattern

- **The tell:** binary media (images, attachments) is stored on disk, with
  only a path string in the database:

  ```sql
  CREATE TABLE Screenshots (
      image_id         SERIAL NOT NULL,
      bug_id           BIGINT UNSIGNED NOT NULL,
      screenshot_path  VARCHAR(100),
      caption          VARCHAR(100),
      PRIMARY KEY (image_id, bug_id),
      FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id)
  );
  ```

- **This becomes the antipattern specifically when it's the unexamined
  default** — "files always belong outside the database" as a rule of
  thumb applied without checking whether the application actually needs
  the guarantees that come with keeping data in the database. It is a
  legitimate design when made deliberately; see §3.
- **The diagnostic isn't in the schema, it's in what's undocumented.**
  Check whether the project has clear answers for: backing up and
  restoring images together with the database; verifying data after a
  restore onto a different server; removing orphaned image files once no
  row references them; access control on who can view/edit an image; and
  what happens to an in-progress image edit if the surrounding operation
  is cancelled. Unclear or missing answers to these point at Phantom
  Files having been chosen carelessly, not deliberately.

---

## 2. Why It Breaks Down

External files sit entirely outside everything the database engine
guarantees:

- **Files don't obey `DELETE`.** Deleting the row that references a file
  doesn't remove the file — nothing does that automatically, so orphaned
  files accumulate unless application code explicitly cleans them up.
- **Files don't obey transaction isolation.** A file change is visible to
  every other client the instant it happens, not when the surrounding
  transaction commits — the opposite of how row changes behave for
  concurrent readers.
- **Files don't obey `ROLLBACK`.** Deleting a file as part of a transaction
  that later rolls back restores the database row but not the file — the
  two diverge permanently, silently.
- **Files don't obey database backup tools.** `mysqldump`, `pg_dump`,
  `rman`, and similar tools back up the database consistently as of one
  point in time; they know nothing about paths stored in a `VARCHAR`
  column. Backing up files needs a separate filesystem-level step, and
  keeping the two backups mutually consistent (same point in time) is hard
  since application code can change files at any moment.
- **Files don't obey SQL access privileges.** `GRANT`/`REVOKE` on tables
  and columns has no effect on files named by a string — access control
  for the actual media has to be reimplemented outside SQL entirely.
- **Files aren't validated as a SQL data type.** The path column is just a
  string; the database can't verify the file exists, that the path is
  well-formed, or keep the path in sync if the file is moved/renamed/
  deleted externally. Every one of those checks becomes application code
  that would otherwise have been the database's job.

---

## 3. Legitimate Uses

Storing media externally is a reasonable, deliberate choice when:

- The database stays much leaner without large binary payloads inflating
  its size — this speeds up ordinary database backups measurably, at the
  cost of a separate file backup step.
- Ad hoc previewing, batch editing, or tooling against the images
  (thumbnailing pipelines, image-processing scripts) is far easier against
  plain files than against `BLOB` columns.
- The transactional guarantees this skill flags as missing genuinely don't
  matter for the workload: single-writer updates, no concurrent access to
  the same image, no rollback requirements, no SQL-level access control
  needed. (E.g., a conference-badge system where each attendee's photo is
  written once by one camera-connected client, with no concurrent access
  or transactional requirements at all — a legitimate case for external
  files.)
- Some databases offer types that keep external-file references more
  transparently managed by the engine — Oracle's `BFILE`, SQL Server's
  `FILESTREAM` — narrowing the gap between the two approaches if you need
  properties of both.

The point isn't "always use BLOB" — it's "decide deliberately, based on
whether your application actually needs the guarantees in §2," rather than
following either convention by reflex.

---

## 4. The Fix: Use BLOB Data Types As Needed

If any of the guarantees in §2 matter for your application, store the media
directly in a `BLOB` (or brand-specific equivalent) column:

```sql
CREATE TABLE Screenshots (
    bug_id            BIGINT UNSIGNED NOT NULL,
    image_id          BIGINT UNSIGNED NOT NULL,
    screenshot_image  BLOB,
    caption           VARCHAR(100),
    PRIMARY KEY (bug_id, image_id),
    FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id)
);
```

This resolves every issue in §2 at once: deleting the row deletes the image;
changes aren't visible to other clients until commit; rollback restores the
prior image state; concurrent updates to the same image are serialized by
ordinary row locking; database backups include the images automatically
and consistently; and SQL privileges on the table/column control access to
the image the same way they control any other data.

- Check the database brand's `BLOB`-family capacity limits before
  assuming it fits — e.g. MySQL's `MEDIUMBLOB` (16MB) vs `LONGBLOB` (4GB),
  Oracle's `LONG RAW` (2GB) vs `BLOB` (4GB). Most images fit comfortably
  within any of these.
- Loading and extracting binary content usually has direct database
  support (e.g. MySQL's `LOAD_FILE()` to import, `SELECT ... INTO
  DUMPFILE` to export) or can be streamed straight through application
  code as a binary HTTP response, without an intermediate file on disk at
  all.

---

## 5. Review Checklist

- Is "store media externally" a deliberate decision backed by an answer
  for backup/restore, orphan cleanup, access control, and rollback
  behavior — or an unexamined default?
- Does deleting a database row currently leave an orphaned file behind,
  with no cleanup mechanism?
- Does the backup process actually capture files and database consistently
  at the same point in time, or could a restore reunite mismatched
  versions (or drop files entirely, as in this chapter's disaster story)?
- Are there concurrency or rollback requirements around image edits that
  external files can't satisfy (visible-before-commit changes, no undo on
  rollback)?
- Does anything need SQL-level access control (`GRANT`/`REVOKE`) over who
  can view or edit the media? External files can't inherit that.
- If none of the above actually matter for this workload, external files
  may be the right, deliberate call — don't convert to BLOB reflexively.

---

## Related guidance

PostgreSQL-specific remedy:

- database-postgresql-blob-bytea-handling
