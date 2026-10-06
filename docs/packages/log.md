---
title: marko/log
description: Logging contracts and formatters -- define how your application logs messages without coupling to a storage backend.
---

Logging contracts and formatters --- define how your application logs messages without coupling to a storage backend. This package provides the `LoggerInterface`, log levels as a backed enum, the `LogRecord` value object, a `LineFormatter`, and typed `LogConfig` for accessing log settings. It contains no storage implementation; install a driver like `marko/log-file` for actual log writing.

## Installation

```bash
composer require marko/log
```

Note: You also need an implementation package such as `marko/log-file`.

## Usage

### Logging Messages

Inject the logger interface and call level-specific methods:

```php
use Marko\Log\Contracts\LoggerInterface;

class OrderService
{
    public function __construct(
        private LoggerInterface $logger,
    ) {}

    public function placeOrder(
        int $orderId,
    ): void {
        $this->logger->info('Order placed', ['order_id' => $orderId]);

        // On failure:
        $this->logger->error('Payment failed for order {order_id}', [
            'order_id' => $orderId,
        ]);
    }
}
```

Context placeholders (`{key}`) are interpolated automatically from the context array.

### Log Levels

Eight severity levels via the `LogLevel` enum (most to least severe):

- `Emergency` --- System unusable
- `Alert` --- Immediate action required
- `Critical` --- Critical conditions
- `Error` --- Runtime errors
- `Warning` --- Exceptional but non-error conditions
- `Notice` --- Normal but significant events
- `Info` --- Interesting events
- `Debug` --- Detailed debug information

Each level has a numeric severity (lower numbers are more severe). Use `meetsThreshold()` to check whether a level is severe enough:

```php
use Marko\Log\LogLevel;

LogLevel::Error->meetsThreshold(LogLevel::Warning); // true (Error is more severe)
LogLevel::Debug->meetsThreshold(LogLevel::Warning); // false
```

### Using a Specific Level

```php
use Marko\Log\LogLevel;

$this->logger->log(LogLevel::Warning, 'Disk space low', [
    'free_mb' => 120,
]);
```

### Configuration

The `LogConfig` class provides typed access to log configuration values:

```php
use Marko\Log\Config\LogConfig;

class MyService
{
    public function __construct(
        private LogConfig $logConfig,
    ) {}

    public function setup(): void
    {
        $driver = $this->logConfig->driver();
        $path = $this->logConfig->path();
        $level = $this->logConfig->level();
        $channel = $this->logConfig->channel();
        $format = $this->logConfig->format();
        $dateFormat = $this->logConfig->dateFormat();
        $maxFiles = $this->logConfig->maxFiles();
        $maxFileSize = $this->logConfig->maxFileSize();
        $escapeNewlines = $this->logConfig->escapeNewlines(); // default true
        $redactKeys = $this->logConfig->redactKeys(); // password, token, secret, ...
        $fileMode = $this->logConfig->fileMode(); // default 0600
        $dirMode = $this->logConfig->dirMode(); // default 0700
    }
}
```

### CLI Commands

| Command | Description |
|---------|-------------|
| `marko log:clear` | Clear log files older than configured `max_files` days |
| `marko log:clear --days=7` | Clear log files older than 7 days |

`log:clear` measures file age against the PSR-20 `ClockInterface` bound by [`marko/clock`](/docs/packages/clock/), so the cutoff is "now" on the same clock the rest of the application uses (a [`FakeClock`](/docs/packages/testing/#fakeclock) in tests).

## Customization

Replace the default log formatter via [Preference](/docs/packages/core/):

```php
use Marko\Core\Attributes\Preference;
use Marko\Log\Contracts\LogFormatterInterface;
use Marko\Log\Formatter\LineFormatter;
use Marko\Log\LogRecord;

#[Preference(replaces: LineFormatter::class)]
class JsonFormatter implements LogFormatterInterface
{
    public function format(
        LogRecord $record,
    ): string {
        return json_encode([
            'level' => $record->level->value,
            'message' => $record->interpolatedMessage(),
            'channel' => $record->channel,
            'datetime' => $record->datetime->format('c'),
            'context' => $record->context,
        ]) . "\n";
    }
}
```

The default `LineFormatter` uses the format `[{datetime}] {channel}.{level}: {message} {context}` and accepts custom format and date format strings via its constructor. The formatter neutralizes untrusted input before it reaches the log file:

- **Control characters** --- in the interpolated message, backslashes are escaped first (`\` becomes `\\`, so a literal `\n` in the input can never be confused with an escaped newline), then every C0 control, DEL, C1 control and the U+2028/U+2029 line terminators are written as escape sequences (`\t`, `\x1b`, `\u0085`, ` `, ...). An ANSI sequence in a value such as `"Login failed for {username}"` therefore cannot repaint an admin's terminal during `tail -f`. In the context JSON the same characters become `\uXXXX` escapes, so it stays valid JSON.
- **Newlines** --- the `escapeNewlines` boolean (default `true`, `log.escape_newlines`) controls whether CR and LF in the message are written as `\r` and `\n`, which prevents log line-injection where one message masquerades as several entries. Disable it only if a downstream log processor requires literal newlines; the other control characters are escaped regardless.
- **Redaction** --- the `redactKeys` list (`log.redact_keys`, default `password`, `password_confirmation`, `token`, `secret`, `api_key`, `authorization`) replaces the value of any matching context key with `[redacted]`. Keys match case-insensitively at any nesting depth of context arrays, and redaction happens before interpolation, so `{token}` in a message is masked too. Pass an empty list to disable it. Objects in the context are not traversed --- convert them to arrays first if they carry secrets.

## API Reference

### LoggerInterface

```php
use Marko\Log\Contracts\LoggerInterface;
use Marko\Log\LogLevel;

public function emergency(string $message, array $context = []): void;
public function alert(string $message, array $context = []): void;
public function critical(string $message, array $context = []): void;
public function error(string $message, array $context = []): void;
public function warning(string $message, array $context = []): void;
public function notice(string $message, array $context = []): void;
public function info(string $message, array $context = []): void;
public function debug(string $message, array $context = []): void;
public function log(LogLevel $level, string $message, array $context = []): void;
```

### LogFormatterInterface

```php
use Marko\Log\Contracts\LogFormatterInterface;
use Marko\Log\LogRecord;

public function format(LogRecord $record): string;
```

### LogLevel

```php
use Marko\Log\LogLevel;

public function severity(): int;
public function meetsThreshold(LogLevel $minimum): bool;
public function upperName(): string;
```

### LogRecord

```php
use Marko\Log\LogRecord;
use Marko\Log\LogLevel;
use DateTimeImmutable;

readonly class LogRecord
{
    public LogLevel $level;
    public string $message;
    public array $context;
    public DateTimeImmutable $datetime;
    public string $channel;

    public function interpolatedMessage(): string;
    public function contextAsJson(): string;
}
```

### LogConfig

```php
use Marko\Log\Config\LogConfig;
use Marko\Log\LogLevel;

public function driver(): string;
public function path(): string;
public function level(): LogLevel;
public function channel(): string;
public function format(): string;
public function dateFormat(): string;
public function maxFiles(): int;
public function maxFileSize(): int;
public function escapeNewlines(): bool; // log.escape_newlines, default true
public function redactKeys(): array; // log.redact_keys, list of context keys to mask
public function fileMode(): int; // log.file_mode, default 0600
public function dirMode(): int; // log.dir_mode, default 0700
```

`redactKeys()` throws `InvalidLogConfigException` for an entry that is not a non-empty string; `fileMode()` and `dirMode()` throw it for a mode outside `0`--`0777`.

### Exceptions

| Exception | Description |
|-----------|-------------|
| `LogException` | Base exception for all log errors --- includes `getContext()` and `getSuggestion()` methods |
| `InvalidLogLevelException` | Thrown when a log level string does not match any valid level |
| `InvalidLogConfigException` | Thrown when `log.redact_keys`, `log.file_mode` or `log.dir_mode` holds an invalid value |
| `LogWriteException` | Thrown when writing to a log file fails (missing directory, permissions) |
