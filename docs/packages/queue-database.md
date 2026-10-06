---
title: marko/queue-database
description: Database queue driver — stores and processes jobs in SQL tables with transaction-safe polling and failed job persistence.
---

Database queue driver --- stores and processes jobs in SQL tables with atomic reservation and failed job persistence. Jobs are stored in a `jobs` table and polled by the worker process. Each job is claimed inside a transaction with the query builder's `lockForUpdate()->skipLocked()` (`FOR UPDATE SKIP LOCKED` on MySQL and PostgreSQL), so concurrent workers skip a job another worker is claiming instead of processing it twice. Crashed reservations (jobs reserved but neither deleted nor released within `queue.retry_after` seconds) are automatically reclaimed on the next poll cycle. Failed jobs are persisted to a `failed_jobs` table for later inspection and retry. Includes migrations for both tables.

Implements `QueueInterface` from [`marko/queue`](/docs/packages/queue/) and requires [`marko/database`](/docs/packages/database/) for the database connection.

## Installation

```bash
composer require marko/queue-database
```

Requires [`marko/database`](/docs/packages/database/) for the database connection. Also requires a non-empty `encryption.key` (via [`marko/encryption`](/docs/packages/encryption/)) because job payloads are HMAC-signed --- see [`marko/queue`](/docs/packages/queue/) for details.

## Usage

### Wiring

Installing the package and a database driver is enough. Its `module.php` binds `QueueInterface` to a `DatabaseQueue` built from your queue config and the driver's `QueryBuilderFactoryInterface`, and binds `FailedJobRepositoryInterface` to `DatabaseFailedJobRepository`. `marko/queue` binds `WorkerInterface`, so `marko queue:work` runs with no extra bindings.

The factory reads these keys from `config/queue.php`:

| Key | Effect on the database driver |
|---|---|
| `queue.queue` | Queue name used by `push()`, `later()` and `pop()` when none is given |
| `queue.retry_after` | Seconds before a reserved job that was neither deleted nor released is reclaimed |
| `queue.max_attempts` | Default attempt limit for jobs that don't set their own `maxAttempts` |

Jobs are always stored in the `jobs` table that the bundled migration creates.

### Running Migrations

Run the included migrations to create the required tables:

```bash
marko migrate
```

This creates:

- `jobs` --- stores pending and reserved jobs
- `failed_jobs` --- stores jobs that exceeded max attempts

### Dispatching and Processing

Use `QueueInterface` as usual --- the database driver handles persistence:

```php
use Marko\Queue\QueueInterface;

public function __construct(
    private readonly QueueInterface $queue,
) {}

public function enqueue(): void
{
    $this->queue->push(new ProcessOrder($orderId));

    // Delay by 5 minutes
    $this->queue->later(
        300,
        new SendFollowUp($orderId),
    );
}
```

Process jobs with the worker:

```bash
marko queue:work
```

### Retries and Attempt Counting

The `jobs.attempts` column is the authoritative attempt count. Every reservation increments it, so it counts attempts that were *started*:

- **A job that throws** is released by the worker. `release()` rewrites the stored payload with the current attempt count. The job is retried until it has been attempted `maxAttempts` times, then moved to `failed_jobs` and deleted from `jobs`.
- **A job whose worker dies** (fatal error, OOM, `SIGKILL`) is never released. Once its reservation is older than `queue.retry_after`, the next `pop()` reclaims it, and the lost run still counts as an attempt. If the reclaimed job has already used all its attempts, `pop()` moves it to `failed_jobs` with the message "exceeded max attempts after worker crash or timeout" and moves on to the next job. A job that always crashes its worker can't loop forever.

`marko queue:retry` resets the attempt count, so a retried job gets its full `maxAttempts` again.

### Time and Testing

`DatabaseQueue` reads the current time from the injected `Psr\Clock\ClockInterface` ([`marko/clock`](/docs/packages/clock/)) for every time-dependent step: `created_at` and `available_at` on push, the `retry_after` reclaim cutoff and `reserved_at` on pop, the availability check in `size()`, the delay on `release()`, and `failedAt` for jobs that exhaust their attempts through crashed reservations. Times are written as `Y-m-d H:i:s` in the clock's timezone (PHP's default timezone for `SystemClock`; bind `new SystemClock('UTC')` to pin it).

Because nothing reads the system time directly, delays and reservation expiry can be tested by moving a [`FakeClock`](/docs/packages/testing/#fakeclock) instead of sleeping:

```php
use Marko\Queue\Database\DatabaseQueue;
use Marko\Testing\Fake\FakeClock;

it('holds a delayed job until it is due', function (): void {
    $clock = new FakeClock('2026-10-05 12:00:00');
    $queue = new DatabaseQueue($connection, $envelope, $failedJobs, $queryBuilderFactory, $clock);
    $queue->later(60, new SendReport());

    $clock->travel('+59 seconds');
    expect($queue->pop())->toBeNull();

    $clock->travel('+1 second');
    expect($queue->pop())->toBeInstanceOf(SendReport::class);
});
```

A reservation becomes reclaimable once `retry_after` seconds have passed since `reserved_at`, inclusive of the boundary second.

### PostgreSQL and Payload Encoding

Payloads use the base64 [envelope format](/docs/packages/queue/#payload-envelope-format). Jobs with private or protected properties therefore store safely in PostgreSQL `TEXT` columns, which reject the NUL bytes that `serialize()` emits. Rows written in the legacy raw format are still read correctly.

## API Reference

### DatabaseQueue

Implements `QueueInterface`. The constructor accepts:

- a `ConnectionInterface` connection
- a `JobEnvelope`
- a `FailedJobRepositoryInterface`, used to fail jobs that exhaust their attempts through crashed reservations
- a `QueryBuilderFactoryInterface`, used to build the locking reservation query. Its builders must use the same connection as the queue, so the lock is taken inside the queue's transaction. The driver bindings already do this.
- a `Psr\Clock\ClockInterface`, the source of every timestamp the queue reads or writes
- an optional table name (`jobs`)
- an optional default queue name
- an optional `retryAfter` timeout in seconds
- an optional default `maxAttempts`

The module factory passes the container's bound clock and sets the last three from `queue.queue`, `queue.retry_after` and `queue.max_attempts`.

`pop()` selects the next available job with `lockForUpdate()->skipLocked()` inside a transaction, then reserves it with an `UPDATE` guarded on `reserved_at`. A job locked by another worker is skipped rather than waited for, so multiple workers can run concurrently without claiming the same job. The connection must implement `TransactionInterface` (the MySQL and PostgreSQL drivers do). Otherwise `pop()` throws `LockException`, because the lock would be released as soon as the `SELECT` finished. Jobs whose `reserved_at` timestamp is older than `retry_after` seconds are treated as crashed and become eligible for re-reservation.

| Method | Description |
|---|---|
| `push(JobInterface $job, ?string $queue = null): string` | Insert a job for immediate processing. Returns the job ID. |
| `later(int $delay, JobInterface $job, ?string $queue = null): string` | Insert a job with a delay in seconds. Returns the job ID. |
| `pop(?string $queue = null): ?JobInterface` | Retrieve and reserve the next available job, or `null` if empty. Increments the `attempts` column, syncs the job's attempt count with earlier unreleased reservations, and moves crash-exhausted jobs to `failed_jobs`. Runs in a transaction (a savepoint inside a caller's transaction). Throws `LockException` on a connection without transactions. |
| `size(?string $queue = null): int` | Count pending (unreserved, available) jobs. |
| `clear(?string $queue = null): int` | Delete all jobs in a queue. Returns the number of deleted rows. |
| `delete(string $jobId): bool` | Delete a specific job by ID. |
| `release(string $jobId, int $delay = 0): bool` | Release a reserved job back to the queue with an optional delay. Rewrites the payload with the persisted attempt count. |

### DatabaseFailedJobRepository

Implements `FailedJobRepositoryInterface`. Accepts a `ConnectionInterface` connection.

| Method | Description |
|---|---|
| `store(FailedJob $failedJob): void` | Persist a failed job record. |
| `all(): array` | Retrieve all failed jobs, most recent first. |
| `find(string $id): ?FailedJob` | Find a failed job by ID, or `null` if not found. |
| `delete(string $id): bool` | Delete a single failed job record. |
| `clear(): int` | Delete all failed job records. Returns the number of deleted rows. |
| `count(): int` | Count total failed jobs. |
