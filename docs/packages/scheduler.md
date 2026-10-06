---
title: marko/scheduler
description: Fluent task scheduler with cron expression support — define recurring tasks in PHP and run them with a single cron entry.
---

Fluent task scheduler with cron expression support --- define recurring tasks in PHP and run them with a single cron entry. Register closures on the `Schedule` with human-readable frequency methods (`daily()`, `hourly()`, `everyFiveMinutes()`) or raw cron expressions. A single system cron entry runs `schedule:run` every minute, and the scheduler determines which tasks are due. No per-task crontab entries needed. Long-running tasks can be protected from overlapping runs, and `schedule:work` runs the scheduler in the foreground where no cron daemon is available.

## Installation

```bash
composer require marko/scheduler
```

## Usage

### Defining Scheduled Tasks

Inject `Schedule` and register tasks in a module's boot callback:

```php title="module.php"
use Marko\Scheduler\Schedule;

return [
    'boot' => function (Schedule $schedule): void {
        $schedule->call(function () {
            // Clean up temp files...
        })->daily()->description('Clean temp files');

        $schedule->call(function () {
            // Send digest emails...
        })->everyFifteenMinutes()->description('Send digest');
    },
];
```

`Schedule` is registered as a singleton by `marko/scheduler`, so the instance your boot callback fills is the same one `schedule:run` and `schedule:work` read from. Boot callbacks run at the end of application initialization, before any command executes.

Give every task a `description()`. It labels the task in command output and is required for overlap protection (see below).

### Frequency Methods

```php
$schedule->call($callback)->everyMinute();
$schedule->call($callback)->everyFiveMinutes();
$schedule->call($callback)->everyTenMinutes();
$schedule->call($callback)->everyFifteenMinutes();
$schedule->call($callback)->everyThirtyMinutes();
$schedule->call($callback)->hourly();
$schedule->call($callback)->daily();
$schedule->call($callback)->weekly();
$schedule->call($callback)->monthly();
```

### Custom Cron Expressions

Use a raw cron expression for full control:

```php
$schedule->call(function () {
    // Runs at 3:30 AM on weekdays
})->cron('30 3 * * 1-5')->description('Weekday report');
```

Supports standard 5-field cron: `minute hour day-of-month month day-of-week`. Fields support:

| Syntax | Example | Meaning |
|--------|---------|---------|
| Wildcard | `*` | Every value |
| Step | `*/5` | Every 5 units |
| Range | `1-5` | Values 1 through 5 |
| Range+step | `1-5/2` | Values 1, 3, 5 |
| List | `1,15,30` | Values 1, 15, and 30 |
| Combined | `1-5,10` | Values 1–5 and 10 |

**Day-of-week:** Both `0` and `7` represent Sunday. When both day-of-month and day-of-week are restricted (neither is `*`), a day matches if *either* field matches (standard cron OR semantics). An invalid or malformed expression throws `InvalidCronExpressionException` loudly.

### Preventing Overlapping Runs

A task that can take longer than its interval --- for example a 90-second import scheduled `everyMinute()` --- would otherwise start a second copy while the first is still running. Call `withoutOverlapping()` to skip the task while a previous run holds its mutex:

```php
$schedule->call(function () {
    // Import the product feed...
})->everyMinute()->description('Import product feed')->withoutOverlapping();
```

While the mutex is held, the run prints `Skipped (still running): Import product feed` and moves on. The mutex is released when the task finishes, including when it throws.

`withoutOverlapping()` takes the number of minutes after which a held mutex is considered stale, for when its holder hung. It defaults to `1440` (24 hours):

```php
$schedule->call($callback)->hourly()->description('Rebuild search index')->withoutOverlapping(90);
```

The mutex is keyed on the task's expression and description, because closures have no stable identity. A task that uses `withoutOverlapping()` without a `description()` makes `schedule:run` and `schedule:work` throw `SchedulerException` before any task runs, whether or not that task is due. An expiry below 1 minute also throws `SchedulerException`.

The default `FileTaskMutex` holds an exclusive `flock()` on `storage/framework/schedule-{hash}` under the project root while the task runs, and records the expiry time in that file. If the process crashes, the operating system releases the lock. A lock that is still held past its expiry is reclaimed by the next run. File locks only protect processes on the same server --- see [Customization](#customization) for multi-server setups.

### Running the Scheduler

Add a single cron entry to your system:

```
* * * * * cd /path/to/project && marko schedule:run
```

The `schedule:run` command checks all registered tasks and executes those that are due. A task that throws is reported as `Failed: {description} - {message}` and the remaining due tasks still run.

| Exit code | Meaning |
|-----------|---------|
| `0` | Every due task succeeded or was skipped as still running (or no tasks were due) |
| `1` | At least one task threw an exception |

The non-zero exit code makes failures visible to cron mail and monitoring tools.

### Running in the Foreground

For local development, or containers without a cron daemon, run the scheduler as a long-lived foreground process:

```bash
marko schedule:work
```

`schedule:work` sleeps until the top of each minute, then runs the tasks due at that minute, using the same logic as `schedule:run`. A failing task is reported and the loop keeps going. Tasks run sequentially in the same process, so if a run takes longer than a minute, the scheduler waits for the next minute boundary rather than catching up on the minutes it missed.

Stop it with `Ctrl+C` (SIGINT) or SIGTERM. With the `pcntl` extension installed, the signal is handled gracefully: the current task finishes, the command prints `Scheduler stopped.` and exits `0`. Without `pcntl`, the signal terminates the process immediately.

### Querying Due Tasks

Programmatically check which tasks are due:

```php
use Marko\Scheduler\Schedule;
use Psr\Clock\ClockInterface;

public function __construct(
    private readonly Schedule $schedule,
    private readonly ClockInterface $clock,
) {}

public function pending(): array
{
    return $this->schedule->dueTasksAt($this->clock->now());
}
```

### Running Due Tasks Programmatically

`ScheduleRunner` is the service both commands use. Call it directly to run the tasks due at a specific time:

```php
use Marko\Core\Command\Output;
use Marko\Scheduler\ScheduleRunner;
use Psr\Clock\ClockInterface;

public function __construct(
    private readonly ScheduleRunner $scheduleRunner,
    private readonly ClockInterface $clock,
) {}

public function runNow(Output $output): bool
{
    $result = $this->scheduleRunner->run($this->clock->now(), $output);

    return !$result->hasFailures();
}
```

### Time and Testing

`schedule:run`, `schedule:work` and `FileTaskMutex` read the current time from the injected `Psr\Clock\ClockInterface` ([`marko/clock`](/docs/packages/clock/)): `schedule:run` runs the tasks due at the clock's time, `schedule:work` sleeps until the clock reaches the next minute, and the mutex writes and checks its expiry against the clock. To test a schedule at a given time, construct the command with a [`FakeClock`](/docs/packages/testing/#fakeclock):

```php
use Marko\Core\Command\Input;
use Marko\Core\Command\Output;
use Marko\Scheduler\Command\RunScheduleCommand;
use Marko\Scheduler\Mutex\FileTaskMutex;
use Marko\Scheduler\ScheduleRunner;
use Marko\Testing\Fake\FakeClock;

it('sends the report on the first of the month', function (): void {
    $clock = new FakeClock('2026-11-01 08:00:00');
    $runner = new ScheduleRunner($schedule, new FileTaskMutex(sys_get_temp_dir() . '/mutex', $clock));
    $command = new RunScheduleCommand($runner, $clock);

    $command->execute(new Input(['marko', 'schedule:run']), new Output(fopen('php://memory', 'r+')));

    expect($reportSender->sent)->toBeTrue();
});
```

## Customization

Overlap protection depends on `TaskMutexInterface`, which `marko/scheduler` binds to `FileTaskMutex`. To share the mutex across several servers, implement the interface on a shared store such as Redis and bind it from your application module, which takes priority over the vendor binding:

```php title="app/scheduling/module.php"
use App\Scheduling\RedisTaskMutex;
use Marko\Scheduler\Mutex\TaskMutexInterface;

return [
    'bindings' => [
        TaskMutexInterface::class => RedisTaskMutex::class,
    ],
];
```

`acquire()` must be atomic and non-blocking, and must treat a mutex older than `$expiresAfterSeconds` as free.

## API Reference

### Schedule

```php
public function call(Closure $callback): ScheduledTask;
public function tasks(): array;
public function dueTasksAt(DateTimeInterface $time): array;
```

### ScheduledTask

```php
public function everyMinute(): self;
public function everyFiveMinutes(): self;
public function everyTenMinutes(): self;
public function everyFifteenMinutes(): self;
public function everyThirtyMinutes(): self;
public function hourly(): self;
public function daily(): self;
public function weekly(): self;
public function monthly(): self;
public function cron(string $expression): self;
public function description(string $description): self;
public function withoutOverlapping(int $expiresAfterMinutes = 1440): self;
public function preventsOverlapping(): bool;
public function getOverlapExpiresAfterMinutes(): ?int;
public function mutexName(): string;
public function getDescription(): ?string;
public function getExpression(): string;
public function getCallback(): Closure;
public function isDue(DateTimeInterface $now): bool;
public function run(): mixed;
```

`mutexName()` throws `SchedulerException` when the task has no description. `withoutOverlapping()` throws `SchedulerException` when the expiry is below 1 minute.

### ScheduleRunner

```php
use Marko\Core\Command\Output;
use Marko\Scheduler\ScheduleRunResult;

public function run(DateTimeInterface $now, Output $output): ScheduleRunResult;
```

### ScheduleRunResult

```php
public int $executed;
public int $failed;
public int $skipped;
public function hasFailures(): bool;
```

### TaskMutexInterface

```php
use Marko\Scheduler\ScheduledTask;

public function acquire(ScheduledTask $task, int $expiresAfterSeconds): bool;
public function release(ScheduledTask $task): void;
public function exists(ScheduledTask $task): bool;
```

### CronExpression

```php
use Marko\Scheduler\CronExpression;
use Marko\Scheduler\Exceptions\InvalidCronExpressionException;

public static function matches(string $expression, DateTimeInterface $time): bool;
```

Throws `InvalidCronExpressionException` if the expression does not have exactly 5 fields or contains characters that cannot be parsed. All fields are validated before any matching occurs.

### Commands

| Command | Description |
|---------|-------------|
| `marko schedule:run` | Run the tasks due this minute; exits `1` if any task failed |
| `marko schedule:work` | Run due tasks at the top of every minute in the foreground until stopped |
