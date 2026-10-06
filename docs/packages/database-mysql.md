---
title: marko/database-mysql
description: MySQL and MariaDB driver for the Marko framework database layer.
---

MySQL and MariaDB driver for the Marko framework database layer. Provides a MySQL-specific connection, query builder, SQL generator, and schema introspector --- all wired automatically when you install the package.

Implements `ConnectionInterface`, `QueryBuilderInterface`, `SqlGeneratorInterface`, and `IntrospectorInterface` from [`marko/database`](/docs/packages/database/).

## Installation

```bash
composer require marko/database-mysql
```

This automatically installs `marko/database` (the interface package) as a dependency.

## Configuration

Create a configuration file at `config/database.php`:

```php title="config/database.php"
<?php

declare(strict_types=1);

return [
    'driver' => 'mysql',
    'host' => $_ENV['DB_HOST'] ?? 'localhost',
    'port' => (int) ($_ENV['DB_PORT'] ?? 3306),
    'database' => $_ENV['DB_DATABASE'] ?? 'marko',
    'username' => $_ENV['DB_USERNAME'] ?? 'root',
    'password' => $_ENV['DB_PASSWORD'] ?? '',
];
```

### Environment Variables

Set these in your `.env` file:

```env
DB_HOST=localhost
DB_PORT=3306
DB_DATABASE=your_database
DB_USERNAME=your_username
DB_PASSWORD=your_password
```

## Driver-Specific Notes

### Generated Primary Keys

MySQL has no `INSERT ... RETURNING`, so `MySqlConnection::supportsReturning()` returns `false` and a repository can't read a key the database generated back. Saving or batch-inserting an entity whose key is marked `#[Column(generated: true)]` but not set throws a `RepositoryException` that names the entity. Set the key in PHP before saving (a UUID from `ramsey/uuid` or `symfony/uid`, for example); a `DEFAULT (UUID())` on the column still fills rows inserted outside the repository. With the key set, the entity saves normally. See [Database-generated keys](/docs/packages/database/#database-generated-keys).

MariaDB 10.5+ supports `INSERT ... RETURNING`, but this driver treats MariaDB like MySQL here: generated keys must be set in PHP on MariaDB too.

### MySQL vs MariaDB

This driver supports both MySQL and MariaDB. CI runs the driver integration tests against **MySQL 8.4** and **MariaDB 11.8** (LTS) on every pull request. Older releases (MySQL 8.0+, MariaDB 10.6+) are expected to work but are not tested. Where the two servers differ:

- **JSON:** MariaDB stores `JSON` as `LONGTEXT` with a `CHECK (json_valid(col))`. The introspector reads such a column back as `json`, so it diffs as unchanged. A column declared by hand as `LONGTEXT CHECK (json_valid(col))` reads as `json` too.
- **Defaults and `ON UPDATE`:** see [Introspected Types and Defaults](#introspected-types-and-defaults).
- **Integer display widths:** MariaDB keeps them in the native type (`int(10) unsigned`, `bigint(20)`), MySQL 8.0.19+ drops them. The native type is not part of the diff.
- **`NOWAIT`:** MySQL reports a lock not acquired with `NOWAIT` as error `3572`, MariaDB as `1205`. Both are `LockTimeoutException`.
- **Shared locks with a modifier:** `sharedLock()->noWait()` and `sharedLock()->skipLocked()` compile to `FOR SHARE`, which MariaDB rejects (see [Locking and upsert on MySQL and MariaDB](#locking-and-upsert-on-mysql-and-mariadb)).

With `innodb_snapshot_isolation=ON` (the default from MariaDB 11.8), a `REPEATABLE READ` transaction that writes a row a concurrent transaction changed after its snapshot fails with error `1020` ("Record has changed since last read"). The driver raises it as `SerializationFailureException`, so `transaction(attempts: ...)` retries it like a deadlock. MySQL never raises `1020` for this case. See [Concurrency Errors and Retries](/docs/packages/database/#concurrency-errors-and-retries).

### Character Set

The default charset is `utf8mb4` which supports the full Unicode range including emojis. This is the recommended setting for new applications.

### Strict Mode

Marko enables MySQL strict mode by default. This ensures data integrity by rejecting invalid data rather than silently truncating or coercing values.

### JSON Columns

MySQL's native `JSON` type is fully supported. Use `type: 'json'` on any `array` or `?array` property:

```php
use Marko\Database\Attributes\Column;

#[Column(type: 'json')]
public array $metadata = [];

#[Column(type: 'json')]
public ?array $settings = null;
```

Values are serialized and deserialized automatically. The root value must be an array --- top-level JSON scalars are not supported. See [marko/database](/docs/packages/database/) for JSON query operators (`whereJsonContains`, arrow-path syntax, etc.) and indexing guidance.

### Column Modifications

When an entity changes a column's type, nullability or default, the migration restates the whole column with `MODIFY COLUMN`. Anything the statement leaves out, MySQL resets, so the generator restates what the database already has wherever the entity doesn't say otherwise:

| Kept from the database | When |
|---|---|
| Length (`VARCHAR(500)`) | The entity declares no `length` |
| Default | The entity declares no `default`, and the new type can hold one (`TEXT`, `BLOB` and `JSON` can't hold a literal, but can hold an expression) |
| Native type: precision, `UNSIGNED`, fractional seconds, `ENUM` values (`decimal(12,4) unsigned`) | The entity keeps the same base type and, for `CHAR`/`VARCHAR`/`BINARY`/`VARBINARY`, the same length |
| Collation (`utf8mb4_bin`) | The column stays a string type. A collation equal to the table default is never pinned |
| `ON UPDATE CURRENT_TIMESTAMP` | The column stays a `TIMESTAMP` or `DATETIME` |

For example, making a `DECIMAL(12,4) UNSIGNED` column nullable with `#[Column(type: 'decimal')]` generates:

```sql
ALTER TABLE `products` MODIFY COLUMN `price` decimal(12,4) unsigned NULL DEFAULT '0.0000'
```

When the entity changes the type, its type wins: `INT UNSIGNED` to `bigint` becomes `BIGINT`. A string column that changes length keeps its collation.

The down migration restates the column exactly as the introspector read it (native type, collation, default and `ON UPDATE`), so `db:rollback` restores the previous schema. A column whose only differences are ones the diff accepts gets no `MODIFY COLUMN` in either direction.

`MODIFY COLUMN` never restates `UNIQUE`: on a column that already has a unique index, it would add a second one. Uniqueness changes are separate statements from the index diff: adding `unique: true` to an existing column generates a `CREATE UNIQUE INDEX` named `<table>_<column>_unique`, and removing it generates `DROP INDEX` for the column's unique index, whatever its name (`email` when an inline `UNIQUE` created it). Both are reversed in the down migration. On a foreign key column, the plain replacement index the diff adds is created before the unique index is dropped (and dropped only after it is restored in down), since InnoDB refuses to drop the last index a foreign key uses. See [Unique Columns on Existing Tables](/docs/packages/database/#unique-columns-on-existing-tables).

### Introspected Types and Defaults

The introspector reports columns in the vocabulary entities use, so an unchanged column diffs as empty:

| MySQL reports | Read as |
|---|---|
| `INT` | `integer` |
| `TINYINT(1)` | `boolean` |
| `CHAR(36)` | `uuid` |
| other types (`BIGINT`, `VARCHAR`, `DECIMAL`, ...) | the lowercase type name |

Literal defaults come back typed: `'0'` is `0` on an integer column, `'0'`/`'1'` are `false`/`true` on a `TINYINT(1)`, and `'0.00'` is `0.0` on a `DECIMAL`, `FLOAT` or `DOUBLE`. The native type (`nativeType`) still holds the full `COLUMN_TYPE`, so a `CHAR(36)` declared as `#[Column(type: 'char', length: 36)]` reads as `uuid` and diffs as changed; declare it as `uuid`.

MariaDB (10.2.7+) reports defaults differently, and the introspector normalizes them: a quoted string default (`'abc'`) is unquoted, the `NULL` it reports for no default is `null`, `current_timestamp()` is `CURRENT_TIMESTAMP` (as a plain string, since MariaDB has no `DEFAULT_GENERATED`), and any other unquoted non-numeric default is an `Expression`. Its `on update current_timestamp()` is read as `CURRENT_TIMESTAMP` (`current_timestamp(3)` as `CURRENT_TIMESTAMP(3)`), and a `LONGTEXT` column with a `json_valid()` check on itself as `json`, with no length or collation.

A type change needs no cast: `MODIFY COLUMN` converts the existing values, and in strict mode a value that doesn't fit (`'abc'` to `INT`) fails the migration.

### Expression Defaults

MySQL 8.0.13+ accepts any expression as a default, as long as it is in parentheses. The generator adds them, except around the `CURRENT_TIMESTAMP` family (`CURRENT_TIMESTAMP`, `CURRENT_TIMESTAMP(6)`, `NOW()`, `LOCALTIMESTAMP`, `LOCALTIME`), which MySQL takes as written:

```php
use Marko\Database\Attributes\Column;
use Marko\Database\Schema\Expression;

#[Column(primaryKey: true, type: 'uuid', default: 'UUID()')]
public string $id;

#[Column(type: 'json', default: new Expression('JSON_ARRAY()'))]
public array $tags = [];

#[Column(type: 'timestamp', default: 'CURRENT_TIMESTAMP')]
public DateTimeImmutable $createdAt;
```

```sql
`id` CHAR(36) NOT NULL DEFAULT (UUID())
`tags` JSON NOT NULL DEFAULT (JSON_ARRAY())
`created_at` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
```

`CURRENT_TIMESTAMP(6)` is passed through as well, but needs a column with the same fractional seconds (`TIMESTAMP(6)`), which only raw DDL creates; an entity's `timestamp` is `TIMESTAMP`. The introspector reads every default MySQL marks `DEFAULT_GENERATED` back as an `Expression` (`(UUID())` comes back as `uuid()`, which the diff treats as the same default). See [Column Defaults](/docs/packages/database/#column-defaults) for the shortcut rules and `Literal`.

MySQL rewrites an expression before storing it: `(CONCAT('a', 'b'))` comes back as `concat(_utf8mb4'a',_utf8mb4'b')`, and `(CURRENT_TIMESTAMP + INTERVAL 1 DAY)` as `(now() + interval 1 day)`. Write the expression as you would in SQL. When an `Expression` default differs from the stored one, `MySqlIntrospector::matchesStoredDefault()` creates a `CREATE TEMPORARY TABLE marko_default_probe` with one column of the real column's type and your expression as its default, reads it back with `SHOW COLUMNS`, and drops it. Creating and dropping a temporary table never commits an open transaction. If what MySQL stored matches the column's default, `db:diff` reports no change. If MySQL rejects the expression, the diff fails with a `MigrationException` naming the column and expression. The connection's user needs the `CREATE TEMPORARY TABLES` privilege. The probe uses the connection's character set, which is what MySQL writes into charset introducers such as `_utf8mb4`, so run `db:diff` with the same connection settings that created the column.

### Partial Indexes

MySQL has no partial indexes. Generating SQL for an `#[Index]` with `where:` throws a `MigrationException` naming the index instead of silently creating a full index. Create the index you need by hand in a migration and list it in `#[Table(unmanagedIndexes: [...])]` (see [Hand-Made Indexes](/docs/packages/database/#hand-made-indexes)).

## Usage

Once configured, the MySQL driver is automatically used when you interact with the database. See [`marko/database`](/docs/packages/database/) for entity definition and repository usage.

```php
use Marko\Database\Connection\ConnectionInterface;

class MyService
{
    public function __construct(
        private ConnectionInterface $connection,
    ) {}

    public function doSomething(): void
    {
        // Connection is automatically MySQL
        $result = $this->connection->query('SELECT * FROM users');
    }
}
```

## API Reference

### Connection

`MySqlConnection` implements `ConnectionInterface` and `TransactionInterface`. It wraps PDO with lazy connection --- the database connection is not established until the first query.

| Method | Description |
|---|---|
| `query(string $sql, array $bindings = []): array` | Execute a query and return all rows as associative arrays |
| `execute(string $sql, array $bindings = []): int` | Execute a statement and return the affected row count |
| `prepare(string $sql): StatementInterface` | Prepare a statement for repeated execution |
| `lastInsertId(): int` | Get the last auto-increment ID |
| `supportsReturning(): bool` | Always `false`: MySQL has no `INSERT ... RETURNING`, so a [database-generated key](/docs/packages/database/#database-generated-keys) must be set in PHP |
| `connect(): void` | Explicitly open the database connection |
| `disconnect(): void` | Close the connection and discard the transaction depth and pending callbacks |
| `isConnected(): bool` | Check whether the connection is open |

### Transactions

`MySqlConnection` also implements `TransactionInterface`, `PendingAfterCommitInterface` and `ResettableInterface`. The driver module registers one shared instance per request (per worker under a long-running runtime), and `TransactionInterface` resolves to that same instance. See [Transactions](/docs/packages/database/#transactions).

| Method | Description |
|---|---|
| `beginTransaction(): void` | Start a transaction, or `SAVEPOINT marko_sp_N` when one is open |
| `commit(): void` | Commit the innermost level (`RELEASE SAVEPOINT` when nested); throws `TransactionException` when none is open |
| `rollback(): void` | Roll back the innermost level (`ROLLBACK TO SAVEPOINT` when nested); throws `TransactionException` when none is open |
| `inTransaction(): bool` | Check whether a transaction is active |
| `transactionLevel(): int` | Number of open levels (0 outside a transaction) |
| `transaction(callable $callback, int $attempts = 1, int\|Closure\|null $backoff = null): mixed` | Execute a callback inside a transaction (a savepoint when nested) --- auto-commits on success, rolls back on exception; the outermost call runs up to `$attempts` times on a deadlock or serialization failure, waiting between attempts as `$backoff` says ([Backoff](/docs/packages/database/#backoff)) |
| `afterCommit(callable $callback): void` | Run the callback after the outermost commit (immediately outside a transaction) |
| `afterRollback(callable $callback): void` | Run the callback if its level rolls back |
| `runPendingAfterCommitCallbacks(): void` | Run the queued `afterCommit()` callbacks without committing (`PendingAfterCommitInterface`); for test helpers such as `RefreshDatabase`, not production code |
| `reset(): void` | Roll back every level left open by a failed request and drop pending callbacks; never opens a connection |

See [Nested Transactions](/docs/packages/database/#nested-transactions) and [After-Commit Callbacks](/docs/packages/database/#after-commit-callbacks).

On a deadlock (`1213`), InnoDB rolls back the whole transaction on the server, savepoints included. A nested level that sees the deadlock closes without sending `ROLLBACK TO SAVEPOINT`, and the `DeadlockException` reaches the outermost `transaction()`, which retries when given `attempts`. A lock wait timeout (`1205`) only fails the statement; the transaction stays open. See [Concurrency Errors and Retries](/docs/packages/database/#concurrency-errors-and-retries).

:::caution
MySQL commits implicitly before any DDL statement (`CREATE`, `ALTER`, `DROP`, `TRUNCATE`, ...). Running one inside a transaction ends it on the server while Marko still counts it as open, and the next `commit()` or `rollback()` fails. Keep schema changes out of transactions.
:::

### Query Builder

`MySqlQueryBuilder` implements `QueryBuilderInterface` with a fluent API:

| Method | Description |
|---|---|
| `table(string $table): static` | Set the target table |
| `select(string ...$columns): static` | Choose columns to return |
| `selectRaw(string $expression, array $bindings = []): static` | Append a raw SQL expression to the SELECT list |
| `where(string $column, string $operator, mixed $value): static` | Add a WHERE condition |
| `orWhere(string $column, string $operator, mixed $value): static` | Add an OR WHERE condition |
| `whereIn(string $column, array $values): static` | Add a WHERE IN condition |
| `whereNull(string $column): static` | Add a WHERE IS NULL condition |
| `whereNotNull(string $column): static` | Add a WHERE IS NOT NULL condition |
| `whereRaw(string $expression, array $bindings = []): static` | Add a raw SQL WHERE condition, AND-combined with other conditions |
| `join(string $table, string $first, string $operator, string $second): static` | Inner join |
| `leftJoin(string $table, string $first, string $operator, string $second): static` | Left join |
| `rightJoin(string $table, string $first, string $operator, string $second): static` | Right join |
| `orderBy(string $column, string $direction = 'ASC'): static` | Order results |
| `orderByRaw(string $expression, string $direction = 'ASC'): static` | Order by a raw SQL expression |
| `limit(int $limit): static` | Limit result count |
| `offset(int $offset): static` | Skip rows |
| `get(): array` | Execute and return all matching rows |
| `first(): ?array` | Execute and return the first row, or `null` |
| `insert(array $data, ?string $primaryKey = null): int` | Insert a row and return the last insert ID. The `$primaryKey` parameter is accepted for interface compatibility but ignored --- MySQL always uses `lastInsertId()`. |
| `update(array $data): int` | Update matching rows and return the affected count |
| `delete(): int` | Delete matching rows and return the affected count |
| `count(?string $column = null): int` | Return the count of matching rows (`COUNT(*)` or `COUNT(column)`) |
| `sum(string $column): int\|float` | Return the sum of a column |
| `avg(string $column): int\|float` | Return the average of a column |
| `min(string $column): int\|float` | Return the minimum value of a column |
| `max(string $column): int\|float` | Return the maximum value of a column |
| `distinct(): static` | Add DISTINCT to the SELECT clause |
| `groupBy(string ...$columns): static` | Add GROUP BY columns |
| `having(string $expression, array $bindings = []): static` | Add a HAVING condition |
| `union(QueryBuilderInterface $query): static` | Append a UNION (deduplicates rows) |
| `unionAll(QueryBuilderInterface $query): static` | Append a UNION ALL (keeps duplicates) |
| `whereJsonContains(string $column, mixed $value): static` | WHERE JSON array contains value |
| `whereJsonExists(string $path): static` | WHERE JSON key/path exists |
| `whereJsonMissing(string $path): static` | WHERE JSON key/path does not exist |
| `raw(string $sql, array $bindings = []): array` | Execute raw SQL |
| `lockForUpdate(): static` | Append `FOR UPDATE` (requires an open transaction) |
| `sharedLock(): static` | Append `LOCK IN SHARE MODE`, or `FOR SHARE` with a modifier (requires an open transaction) |
| `skipLocked(): static` | Append `SKIP LOCKED` to the lock |
| `noWait(): static` | Append `NOWAIT` to the lock |
| `upsert(array $rows, array $uniqueBy, ?array $update = null): int` | `INSERT ... ON DUPLICATE KEY UPDATE col = VALUES(col)`; returns the affected-row count (2 per updated row) |

#### Locking and upsert on MySQL and MariaDB

- `sharedLock()` compiles to `LOCK IN SHARE MODE`, which MySQL and MariaDB both accept. `LOCK IN SHARE MODE` takes no modifiers, so `sharedLock()->skipLocked()` / `->noWait()` compiles to `FOR SHARE SKIP LOCKED` / `FOR SHARE NOWAIT`, which needs MySQL 8.0+. MariaDB does not support `FOR SHARE`. `SKIP LOCKED` needs MariaDB 10.6+.
- `upsert()` resolves a conflict against **any** unique index or primary key the row violates, not only the `$uniqueBy` columns, which shape only the default update list. An empty update list compiles to a no-op assignment (`col = col`) instead of `INSERT IGNORE`, so unrelated errors still surface.
- `upsert()` uses `VALUES(col)` rather than the row alias added in MySQL 8.0.19, because MariaDB supports only `VALUES()`. MySQL 8.0.20+ reports `VALUES()` as deprecated but still runs it.

### MySqlExceptionTranslator

Turns a `PDOException` raised by `query()`, `execute()`, `prepare()` or `MySqlStatement::execute()` into a typed exception from `marko/database`, keyed on the server error number (MySQL reports every integrity violation as SQLSTATE `23000`): `1062` unique; `1451`, `1452`, `1216`, `1217` foreign key; `1048`, `1364` not null; `3819` (MySQL) and `4025` (MariaDB) check; `1213` `DeadlockException` (InnoDB also reports most serialization conflicts this way); `1020` `SerializationFailureException` (MariaDB with `innodb_snapshot_isolation=ON`); `1205` (lock wait timeout) and `3572` (`NOWAIT`) `LockTimeoutException`; anything else `QueryException`. Failed `BEGIN`, `COMMIT`, `SAVEPOINT`, `RELEASE SAVEPOINT` and `ROLLBACK` statements are translated the same way. The constraint, table and column are parsed from the server message. The duplicate value in a `1062` message is never copied. `MySqlConnection` and `MySqlStatement` take it as an optional constructor argument (`exceptionTranslator`). See [Query and Constraint Exceptions](/docs/packages/database/#query-and-constraint-exceptions) and [Concurrency Errors and Retries](/docs/packages/database/#concurrency-errors-and-retries).

| Method | Description |
|---|---|
| `translate(PDOException $exception, string $sql, array $bindings): QueryException` | Map a driver error to the typed exception, keeping the `PDOException` as `getPrevious()` |

### MySqlConnectionFactory

Implements `ConnectionFactoryInterface`. Creates `MySqlConnection` instances from a `DatabaseConfig`. Used by `marko/database-readwrite` to build per-connection instances for the write primary and each read replica.

| Method | Description |
|---|---|
| `make(DatabaseConfig $config): ConnectionInterface` | Create and return a new `MySqlConnection` for the given config |

The factory hands every connection it makes the container-bound `TransactionBackoff`, so the write primary and the replicas wait between `transaction()` retries the same way as the default connection.

### SQL Generator

`MySqlGenerator` implements `SqlGeneratorInterface` --- produces MySQL-specific DDL from schema diffs (used by the migration system).

| Abstract Type | MySQL Type |
|---|---|
| `integer` / `int` | `INT` |
| `bigint` | `BIGINT` |
| `smallint` | `SMALLINT` |
| `tinyint` | `TINYINT` |
| `string` | `VARCHAR(n)` (default 255) |
| `text` | `TEXT` |
| `boolean` / `bool` | `TINYINT(1)` |
| `datetime` | `DATETIME` |
| `date` | `DATE` |
| `time` | `TIME` |
| `timestamp` | `TIMESTAMP` |
| `decimal` | `DECIMAL(10,2)` |
| `float` | `FLOAT` |
| `double` | `DOUBLE` |
| `blob` / `binary` | `BLOB` |
| `json` | `JSON` |

### Introspector

`MySqlIntrospector` implements `IntrospectorInterface` and `ExpressionDefaultMatcherInterface` --- reads the live database schema via `information_schema` for use by the migration diff calculator.

| Method | Description |
|---|---|
| `getTables(): array` | List all table names in the database |
| `getTable(string $name): ?Table` | Get a full `Table` schema object (columns, indexes, foreign keys) |
| `tableExists(string $name): bool` | Check whether a table exists |
| `getColumns(string $table): array` | Get column definitions for a table with abstract type names and typed defaults (see [Introspected Types and Defaults](#introspected-types-and-defaults)), including each column's native type (`nativeType`), a collation that differs from the table default (`collation`) and its `ON UPDATE` expression (`onUpdateExpression`). A `DEFAULT_GENERATED` default is returned as an `Expression`, and a string literal that reads like a function (`'now()'`) as a `Literal` |
| `getIndexes(string $table): array` | Get index definitions for a table, single-column unique indexes included |
| `getForeignKeys(string $table): array` | Get foreign key definitions for a table |
| `getPrimaryKey(string $table): array` | Get primary key column names for a table |
| `matchesStoredDefault(string $table, string $column, Expression $expression): bool` | Whether the column would store its current default if it were declared with `$expression`, checked on a temporary table that is dropped again (see [Expression Defaults](#expression-defaults)). Throws `MigrationException` when MySQL rejects the expression |
