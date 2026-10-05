---
title: marko/clock
description: PSR-20 system clock --- inject ClockInterface instead of calling time() so time-dependent code can be tested deterministically.
---

PSR-20 system clock --- inject `ClockInterface` instead of calling `time()` so time-dependent code can be tested deterministically. The package binds the standard `Psr\Clock\ClockInterface` to `SystemClock`, which reads the system time. There is no Marko-specific clock interface: PSR-20 is already a single explicit method, and depending on it keeps your code compatible with any other PSR-20 clock (Symfony's or Lcobucci's, for example). In tests, swap in [`FakeClock`](/docs/packages/testing/#fakeclock) from `marko/testing`.

## Installation

```bash
composer require marko/clock
```

The module binds `ClockInterface` to `SystemClock` as a singleton. There is no configuration file.

## Usage

### Injecting the Clock

Type-hint `Psr\Clock\ClockInterface` in a constructor and call `now()` wherever you would have called `time()` or `new DateTimeImmutable()`:

```php
use DateTimeImmutable;
use Psr\Clock\ClockInterface;

class InvitationService
{
    public function __construct(
        private ClockInterface $clock,
    ) {}

    public function expiresAt(): DateTimeImmutable
    {
        return $this->clock->now()->modify('+7 days');
    }

    public function isExpired(Invitation $invitation): bool
    {
        return $invitation->expiresAt < $this->clock->now();
    }
}
```

For a Unix timestamp, use `$this->clock->now()->getTimestamp()`.

### Timezone

By default `SystemClock` returns times in PHP's default timezone (`date_default_timezone_get()`), read each time `now()` is called. To pin a timezone, construct the clock yourself and bind it in your module:

```php title="app/core/module.php"
use Marko\Clock\SystemClock;
use Marko\Core\Container\ContainerInterface;
use Psr\Clock\ClockInterface;

return [
    'bindings' => [
        ClockInterface::class => fn (ContainerInterface $container): ClockInterface => new SystemClock('UTC'),
    ],
];
```

The constructor accepts a `DateTimeZone` or a timezone name. An unknown name throws `DateInvalidTimeZoneException`.

### Testing

Pass a `FakeClock` wherever the code under test expects a `ClockInterface`. It is frozen until you move it, so boundaries can be checked exactly:

```php
use Marko\Testing\Fake\FakeClock;

it('expires an invitation after seven days', function (): void {
    $clock = new FakeClock('2026-01-01 12:00:00 UTC');
    $service = new InvitationService($clock);
    $invitation = new Invitation(expiresAt: $service->expiresAt());

    $clock->travel('+7 days');
    expect($service->isExpired($invitation))->toBeFalse();

    $clock->travel('+1 second');
    expect($service->isExpired($invitation))->toBeTrue();
});
```

### Packages Using the Clock

These packages take an injected `ClockInterface` and so require `marko/clock`:

- [`marko/session`](/docs/packages/session/) --- session cookie expiry
- [`marko/session-file`](/docs/packages/session-file/) and [`marko/session-database`](/docs/packages/session-database/) --- `last_activity` and garbage collection
- [`marko/authentication-token`](/docs/packages/authentication-token/) --- personal access token expiry
- [`marko/webhook`](/docs/packages/webhook/) --- signature timestamps, the freshness window, and attempt times

## API Reference

### SystemClock

```php
use Marko\Clock\SystemClock;

public function __construct(DateTimeZone|string|null $timezone = null);
public function now(): DateTimeImmutable;
```

### ClockInterface

```php
use Psr\Clock\ClockInterface;

public function now(): DateTimeImmutable;
```

## Related Packages

- [`marko/testing`](/docs/packages/testing/) --- `FakeClock` for deterministic tests
