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

### MySQL vs MariaDB

This driver supports both MySQL 8.0+ and MariaDB 10.5+. Both are tested and fully supported.

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
| Default | The entity declares no `default`, and the new type can hold one (`TEXT`, `BLOB` and `JSON` can't) |
| Native type: precision, `UNSIGNED`, fractional seconds, `ENUM` values (`decimal(12,4) unsigned`) | The entity keeps the same base type and, for `CHAR`/`VARCHAR`/`BINARY`/`VARBINARY`, the same length |
| Collation (`utf8mb4_bin`) | The column stays a string type. A collation equal to the table default is never pinned |
| `ON UPDATE CURRENT_TIMESTAMP` | The column stays a `TIMESTAMP` or `DATETIME` |

For example, making a `DECIMAL(12,4) UNSIGNED` column nullable with `#[Column(type: 'decimal')]` generates:

```sql
ALTER TABLE `products` MODIFY COLUMN `price` decimal(12,4) unsigned NULL DEFAULT '0.0000'
```

When the entity changes the type, its type wins: `INT UNSIGNED` to `bigint` becomes `BIGINT`. A string column that changes length keeps its collation.

The down migration restates the column exactly as the introspector read it (native type, collation, default and `ON UPDATE`), so `db:rollback` restores the previous schema. A column whose only differences are ones the diff accepts gets no `MODIFY COLUMN` in either direction.

`MODIFY COLUMN` never restates `UNIQUE`: on a column that already has a unique index, it would add a second one. The index diff handles uniqueness.

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
| `transaction(callable $callback, int $attempts = 1): mixed` | Execute a callback inside a transaction (a savepoint when nested) --- auto-commits on success, rolls back on exception; the outermost call runs up to `$attempts` times on a deadlock |
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

Turns a `PDOException` raised by `query()`, `execute()`, `prepare()` or `MySqlStatement::execute()` into a typed exception from `marko/database`, keyed on the server error number (MySQL reports every integrity violation as SQLSTATE `23000`): `1062` unique; `1451`, `1452`, `1216`, `1217` foreign key; `1048`, `1364` not null; `3819` (MySQL) and `4025` (MariaDB) check; `1213` `DeadlockException` (InnoDB also reports serialization conflicts this way); `1205` (lock wait timeout) and `3572` (`NOWAIT`) `LockTimeoutException`; anything else `QueryException`. Failed `BEGIN`, `COMMIT`, `SAVEPOINT`, `RELEASE SAVEPOINT` and `ROLLBACK` statements are translated the same way. The constraint, table and column are parsed from the server message. The duplicate value in a `1062` message is never copied. `MySqlConnection` and `MySqlStatement` take it as an optional last constructor argument. See [Query and Constraint Exceptions](/docs/packages/database/#query-and-constraint-exceptions) and [Concurrency Errors and Retries](/docs/packages/database/#concurrency-errors-and-retries).

| Method | Description |
|---|---|
| `translate(PDOException $exception, string $sql, array $bindings): QueryException` | Map a driver error to the typed exception, keeping the `PDOException` as `getPrevious()` |

### MySqlConnectionFactory

Implements `ConnectionFactoryInterface`. Creates `MySqlConnection` instances from a `DatabaseConfig`. Used by `marko/database-readwrite` to build per-connection instances for the write primary and each read replica.

| Method | Description |
|---|---|
| `make(DatabaseConfig $config): ConnectionInterface` | Create and return a new `MySqlConnection` for the given config |

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

`MySqlIntrospector` implements `IntrospectorInterface` --- reads the live database schema via `information_schema` for use by the migration diff calculator.

| Method | Description |
|---|---|
| `getTables(): array` | List all table names in the database |
| `getTable(string $name): ?Table` | Get a full `Table` schema object (columns, indexes, foreign keys) |
| `tableExists(string $name): bool` | Check whether a table exists |
| `getColumns(string $table): array` | Get column definitions for a table, including each column's native type (`nativeType`), a collation that differs from the table default (`collation`) and its `ON UPDATE` expression (`onUpdateExpression`) |
| `getIndexes(string $table): array` | Get index definitions for a table |
| `getForeignKeys(string $table): array` | Get foreign key definitions for a table |
| `getPrimaryKey(string $table): array` | Get primary key column names for a table |
