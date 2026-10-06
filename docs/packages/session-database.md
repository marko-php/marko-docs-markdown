---
title: marko/session-database
description: Database session driver — stores session data in a SQL table for shared access across multiple application servers.
---

Database session driver --- stores session data in a SQL table for shared access across multiple application servers. Sessions are stored in a `sessions` table with columns for the session ID, serialized payload, and last activity timestamp. This enables session sharing across multiple web servers behind a load balancer. Garbage collection deletes rows where `last_activity` exceeds the configured session lifetime.

Implements `SessionHandlerInterface` from [`marko/session`](/docs/packages/session/) and requires [`marko/database`](/docs/packages/database/) for the database connection.

## Installation

```bash
composer require marko/session-database
```

Requires [`marko/database`](/docs/packages/database/) for the database connection.

## Usage

Installing `marko/session-database` is all that is required to activate database-backed sessions. The package registers `DatabaseSessionHandler` as the `SessionHandlerInterface` implementation, binds `SessionInterface` to `Session` as a singleton, and adds `SessionMiddleware` globally --- no manual configuration is needed.

Use `SessionInterface` as usual:

```php
use Marko\Session\Contracts\SessionInterface;

public function __construct(
    private readonly SessionInterface $session,
) {}

public function handle(): void
{
    $this->session->start();
    $this->session->set('cart_id', $cartId);
    $this->session->save();
}
```

### Creating the Sessions Table

The package ships a `DatabaseSession` entity (`Marko\Session\Database\Entity\DatabaseSession`) that owns the `sessions` table schema. Create the table with [`marko db:migrate`](/docs/packages/database/):

```bash
marko db:migrate
```

In development, `db:migrate` generates a migration for the table in `database/migrations/` and applies it. Commit that migration; `db:migrate` on staging and production applies it, because generation only runs in development. The table has these columns on MySQL, MariaDB and PostgreSQL:

| Column | Type | Notes |
|---|---|---|
| `id` | `VARCHAR(128)` | Primary key |
| `payload` | `TEXT` | Serialized session data |
| `last_activity` | `INT` | Unix timestamp of the last write or refresh |

The entity only owns the schema: `DatabaseSessionHandler` reads and writes the table with its own SQL. Because the table belongs to an entity, `db:diff` reports drift in it and [`TruncateDatabase`](/docs/packages/testing/#truncatedatabase) empties it between tests.

A `sessions` table created from the SQL these docs gave before the entity existed already matches it, so `db:migrate` generates nothing. For a table with a different shape (for example `id VARCHAR(255)`), `db:migrate` in development generates a migration that changes it to the columns above, and `db:diff` shows that change first. Review the migration before you commit it.

### Garbage Collection

Remove expired sessions via CLI:

```bash
marko session:gc
```

This deletes rows where `last_activity` is older than the configured session lifetime.

A session cookie only resumes a row that exists and is within the lifetime (see [Strict session IDs](/docs/packages/session/#strict-session-ids)). Replaying an unknown or expired cookie inserts nothing, and a resumed session that wasn't modified only updates `last_activity`.

### Read Replicas

With [`marko/database-readwrite`](/docs/packages/database-readwrite/), `read()` and `validateId()` run on the primary through [`onPrimary()`](/docs/packages/database-readwrite/#reads-that-must-hit-the-primary), never on a replica. Logout and ID regeneration delete the old row on the primary, and a lagging replica would otherwise still return it, letting a replayed old cookie resume the logged-out session.

## API Reference

### DatabaseSessionHandler

Implements `SessionHandlerInterface`. Takes a `ConnectionInterface` connection, the `SessionConfig` (for the lifetime) and a PSR-20 clock; all three are autowired.

| Method | Description |
|---|---|
| `open(string $path, string $name): bool` | Open the session store. Returns `true`. |
| `close(): bool` | Close the session store. Returns `true`. |
| `read(string $id): string\|false` | Read session data by ID. Returns an empty string if the session does not exist. |
| `write(string $id, string $data): bool` | Write session data using a single atomic upsert. MySQL uses `ON DUPLICATE KEY UPDATE`; PostgreSQL uses `ON CONFLICT (id) DO UPDATE`. No separate delete-then-insert. |
| `destroy(string $id): bool` | Delete a session by ID. |
| `gc(int $max_lifetime): int\|false` | Delete sessions where `last_activity` is older than `max_lifetime` seconds. Returns the number of deleted rows. |
| `validateId(string $id): bool` | `SELECT 1 FROM sessions WHERE id = ? AND last_activity >= ?`: `true` only for a row within the configured `lifetime`. Unknown and expired IDs get a fresh session; expired rows are left for `gc()`. |
| `updateTimestamp(string $id, string $data): bool` | `UPDATE sessions SET last_activity = ? WHERE id = ?` for a resumed session whose data didn't change. Never rewrites the payload and never inserts a row. |
