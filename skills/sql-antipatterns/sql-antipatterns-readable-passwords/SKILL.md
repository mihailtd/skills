---
name: sql-antipatterns-readable-passwords

description: Detects the Readable Passwords antipattern — storing passwords in plain text (or any reversible form) so they can be emailed back or compared directly in SQL — which exposes them via network capture, query logs, and backups, and guides toward a salted cryptographic hash, password reset instead of recovery, and keeping plaintext out of SQL statements entirely.
---

# SQL Antipatterns — Readable Passwords

This skill helps developers recognize when a password-recovery feature
implies passwords are stored in a readable (plain-text or reversibly
encoded) form, and guides them toward salted hashing, password reset
instead of recovery, and keeping plaintext passwords out of SQL statements
entirely.

---

## 1. Recognize the Antipattern

- **The tell:** any feature that can show a user their original password
  back — "email me my password," a support rep reading a password over
  the phone — necessarily means the password is stored in plain text or
  some other reversible form. There is no way to build password *recovery*
  (as opposed to *reset*) without that being true.

  ```sql
  CREATE TABLE Accounts (
      account_id   SERIAL PRIMARY KEY,
      account_name VARCHAR(20) NOT NULL,
      email        VARCHAR(100) NOT NULL,
      password     VARCHAR(30) NOT NULL
  );
  ```

- If a password can be read for a legitimate purpose (support staff
  lookup, "forgot password" email), assume it can be read for an
  illegitimate one too — the storage mechanism doesn't distinguish who's
  asking.

---

## 2. Why It Breaks Down

Storing (or transmitting) a password in a form that can be read back
exposes it at every point that data passes through:

- **Network interception.** An SQL statement carrying a plaintext password
  — for insert, update, or comparison during login — is readable by anyone
  who can capture the traffic (e.g. with Wireshark/tcpdump) if the
  connection isn't encrypted in transit.
- **Query logs.** Database servers commonly log executed statements.
  Anyone with access to those logs — which may have looser access control
  than the database itself — can read every password that ever appeared
  in a logged statement.
- **Backups.** Database backup files and backup media often get *less*
  strict access control than the live database, making them an easier
  target for the same data.
- **Authentication-by-comparison leaks the same way.** Comparing user
  input to a stored plaintext password (`WHERE password = 'opensesame'`)
  puts the submitted password in the SQL statement too — every point above
  applies to login attempts, not just storage.
- **Lumping account-lookup and password-check into one condition hides
  useful signal.** `WHERE account_name = 'bill' AND password = 'opensesame'`
  can't tell you whether the account doesn't exist or the password was
  wrong. If you want to detect repeated failed attempts against a real
  account (a plausible intrusion signal) and respond differently than to
  someone probing nonexistent usernames, this query shape throws that
  distinction away.
- **Emailing a plaintext password compounds all of the above.** Email is
  routed across networks and infrastructure you don't control, and can be
  intercepted, logged, or stored well beyond the sending and receiving
  servers you might trust — "we used TLS for the mail transfer" doesn't
  contain the exposure once the message exists.

---

## 3. Legitimate Uses

- If your application is itself a *client* to a third-party service and
  needs to present that service's credentials on the user's behalf, it
  genuinely needs the credential in a readable form — store it with
  reversible encryption your application controls, not plain text, and
  scope access to it tightly. This is a fundamentally different case from
  storing a password *your own application* uses to authenticate its own
  users.
- There's a real distinction between **identification** (claiming who you
  are) and **authentication** (proving it). If an application's threat
  model genuinely doesn't require defeating a skilled, motivated attacker
  — e.g. a small internal tool used only by known, trusted people — a
  simpler identification-only scheme may be an adequate, deliberate
  choice, not negligence. The risk: applications drift beyond their
  original scope. Before such a tool becomes reachable outside its
  original trust boundary (exposed past a firewall, opened to more users),
  get it reviewed by someone who can evaluate it against its new exposure.
- If you're asked to build password *recovery* (not reset), it's a
  reasonable, professional response to push back and explain the risk,
  then offer reset as a functionally equivalent alternative — this is
  analogous to any engineer flagging an unsafe design before building it,
  not overstepping scope.

---

## 4. The Fix: Store a Salted Hash, Reset Instead of Recover

### Hash, don't store plaintext

Encode the password with a one-way cryptographic hash before it's ever
stored — the hash can validate a login attempt without the original
password ever needing to be recoverable:

```sql
CREATE TABLE Accounts (
    account_id    SERIAL PRIMARY KEY,
    account_name  VARCHAR(20),
    email         VARCHAR(100) NOT NULL,
    password_hash CHAR(64) NOT NULL  -- SHA-256 hex digest is a fixed 64 chars
);

INSERT INTO Accounts (account_id, account_name, email, password_hash)
VALUES (123, 'billkarwin', 'bill@example.com', SHA2('xyzzy', 256));

SELECT CASE WHEN password_hash = SHA2('xyzzy', 256) THEN 1 ELSE 0 END
FROM Accounts WHERE account_id = 123;
```

Use a hash algorithm currently considered strong (SHA-256/SHA-3 family at
minimum for general hashing; see below for password-specific algorithms).
MD5 and SHA-1 are both cryptographically broken for this purpose — don't
use them for anything security-sensitive, even though they still have
legitimate non-security uses elsewhere.

To disable an account without a dedicated "disabled" flag, you can
overwrite `password_hash` with a value the hash function could never
produce (e.g. containing non-hex characters) — but prefer an explicit
disabled/active column where the schema supports it; it's clearer intent
than an unguessable-hash trick.

### Add a salt to defeat dictionary/rainbow-table attacks

A bare hash is still vulnerable: an attacker who obtains your hash column
can precompute hashes of common passwords and match them against your
table in bulk. A per-password random salt, concatenated with the password
before hashing, defeats this — the same password produces a different
hash for every user, so no precomputed table applies:

```sql
CREATE TABLE Accounts (
    account_id    SERIAL PRIMARY KEY,
    account_name  VARCHAR(20),
    email         VARCHAR(100) NOT NULL,
    password_hash CHAR(64) NOT NULL,
    salt          BINARY(8) NOT NULL
);

INSERT INTO Accounts (account_id, account_name, email, password_hash, salt)
VALUES (123, 'billkarwin', 'bill@example.com',
        SHA2(CONCAT('xyzzy', '-0xT!sp9'), 256), '-0xT!sp9');
```

Generate the salt randomly per password, at least 8 bytes, and don't
restrict it to printable characters.

### Compute the hash in application code, not in the SQL statement

Even salted and hashed, a plaintext password sent as part of a SQL
statement (`SHA2('xyzzy', 256)`) is still visible to anyone who intercepts
that statement or reads it from a log — the hash was computed
server-side from an exposed plaintext value. Instead, fetch the salt,
compute the hash client-side, and send only the resulting hash to the
database for comparison — the plaintext password never appears in SQL at
all:

```python
query = "SELECT password_hash, salt FROM Accounts WHERE account_name = %s"
cursor.execute(query, (name,))
stored_hash, salt = cursor.fetchone()
input_hash = hashlib.sha256(f"{password}{salt.decode()}".encode()).hexdigest()
if input_hash == stored_hash:
    ...
```

Also use TLS for the connection carrying the login form submission itself
(browser to application server) — hashing client-side in the browser
before submission is a defense-in-depth option worth considering on top
of transport encryption, not instead of it.

### Reset, don't recover

Once passwords are hashed, recovery is impossible by construction — which
is the point. Offer reset instead:

- **Temporary system-generated password**, emailed, expiring quickly. The
  application knows the plaintext only transiently (to send the email);
  only its hash is ever stored.
- **Single-use, expiring reset token**, stored server-side with an
  expiration and tied to one account:

  ```sql
  CREATE TABLE PasswordResetRequest (
      account_id BIGINT UNSIGNED PRIMARY KEY,
      token      CHAR(32) NOT NULL,
      expiration TIMESTAMP NOT NULL,
      FOREIGN KEY (account_id) REFERENCES Accounts(account_id)
  );
  ```

  Email only the token (as a link), never a password. Validate that the
  token matches, hasn't expired, and is tied to the account being reset,
  before allowing a new password to be set. Force a password change before
  any other action once the reset flow completes.

Either way, send the reset email/SMS only to the contact info already on
file for the account — never to an address supplied fresh in the request
— so an illegitimate reset attempt only notifies the real account owner,
not the attacker.

### Prefer password-specific hashing algorithms over general-purpose ones

General cryptographic hashes (SHA-256, SHA-3) are fast — which is exactly
the wrong property for password hashing, since it's what makes brute-force
guessing cheap for an attacker. Purpose-built password-hashing algorithms
(Argon2, PBKDF2, Bcrypt) are deliberately slow/memory-hard and are the
current recommended choice; check current NIST guidance periodically, as
recommended algorithms shift over time.

---

## 5. Storing Hash Output: Use a Fixed-Length Type, Not VARCHAR

A hash function always produces a fixed-length output regardless of input
length — so there's no reason to store it in a variable-length `VARCHAR`
(commonly over-allocated as `VARCHAR(255)` "just in case"). Use a
fixed-length type sized to the actual output:

| Algorithm | Bits | Hex digits (`CHAR`) | Base64 (`CHAR`) | Raw bytes (`BINARY`) |
|---|---|---|---|---|
| MD5 | 128 | `CHAR(32)` | `CHAR(22)` | `BINARY(16)` |
| SHA-1 | 160 | `CHAR(40)` | `CHAR(27)` | `BINARY(20)` |
| SHA-256 | 256 | `CHAR(64)` | `CHAR(43)` | `BINARY(32)` |
| SHA-512 | 512 | `CHAR(128)` | `CHAR(86)` | `BINARY(64)` |

- A hex-digit string uses two characters per byte; storing the same value
  as raw binary (`BINARY`) halves the storage. Base64 splits the
  difference — more compact than hex (4 characters per 3 bytes vs. 2
  characters per byte) while staying printable/human-readable, unlike raw
  binary.
- Password-hashing algorithms with configurable parameters (Argon2,
  PBKDF2, Bcrypt) encode the algorithm, version, options, and salt
  alongside the hash in one string (e.g. the PHC format:
  `$argon2i$v=19$m=4096,t=3,p=1$<salt>$<hash>`) — size the column to that
  full encoded string's actual length for your chosen parameters, not just
  the raw hash length, and don't try to parse it apart into separate
  columns; compare the whole encoded string as one unit.

Getting this right isn't just tidiness — a correctly-sized fixed-length
column (and any index on it) uses meaningfully less storage than an
oversized `VARCHAR` at scale.

---

## 6. Review Checklist

- Does any feature let a password be retrieved and shown/emailed back to a
  user, implying it's stored in a readable form?
- Are login/password-change SQL statements ever built with the plaintext
  password interpolated directly into the query, rather than a
  precomputed hash?
- Is the password hash salted per-user, or would two users with the same
  password produce the same stored hash (vulnerable to bulk dictionary
  matching)?
- Is the hashing algorithm a fast general-purpose hash (MD5/SHA-family)
  rather than a purpose-built, deliberately slow password-hashing
  algorithm (Argon2/PBKDF2/Bcrypt)?
- Does "forgot password" send a reset link/temporary credential to the
  address already on file, rather than accepting a fresh destination in
  the request?
- Is the hash/salt column type sized to the algorithm's actual fixed
  output length, rather than an oversized `VARCHAR`?

---

## Related guidance

PostgreSQL-specific remedy:

- database-postgresql-pgcrypto-encryption
