---
title: marko/core
description: The foundation of Marko — provides dependency injection, modules, plugins, events, and preferences so you can extend any class without modifying its source.
---

The foundation of Marko — provides dependency injection, modules, plugins, events, and preferences so you can extend any class without modifying its source. Core gives you the extensibility primitives: replace any class with `#[Preference]`, modify any method with `#[Before]`/`#[After]` plugins, react to events with `#[Observer]`. Everything is a module, and modules are discovered automatically from `vendor/`, `modules/`, and `app/`.

## Installation

```bash
composer require marko/core
```

Note: Most applications install this via a metapackage or implementation package.

## Usage

### Replacing Classes with Preferences

Override any class globally without touching its source:

```php
use Marko\Core\Attributes\Preference;

#[Preference(replaces: OriginalService::class)]
class MyService extends OriginalService
{
    public function doSomething(): string
    {
        // Your implementation
        return 'custom behavior';
    }
}
```

Anywhere `OriginalService` is injected, `MyService` is provided instead.

### Modifying Methods with Plugins

Intercept method calls without replacing the whole class:

```php
use Marko\Core\Attributes\Plugin;
use Marko\Core\Attributes\Before;
use Marko\Core\Attributes\After;

#[Plugin(target: PaymentService::class)]
class PaymentValidationPlugin
{
    #[Before]
    public function charge(
        float $amount,
    ): null|array {
        // Modify input — return an array to replace the arguments
        return [$amount * 1.1]; // Add 10% fee
    }
}

#[Plugin(target: PaymentService::class)]
class PaymentAuditPlugin
{
    #[After]
    public function charge(
        Receipt $result,
    ): Receipt {
        // Modify output
        return $result->withTax();
    }
}
```

### Reacting to Events

Decouple "something happened" from "react to it":

```php
use Marko\Core\Attributes\Observer;
use Marko\Core\Event\Event;

#[Observer(event: UserCreatedEvent::class)]
class SendWelcomeEmail
{
    public function handle(
        UserCreatedEvent $event,
    ): void {
        $user = $event->user;
        // Send email...
    }
}
```

Dispatch events from anywhere:

```php
$this->eventDispatcher->dispatch(new UserCreatedEvent(user: $user));
```

### Creating Modules

Create a directory in `app/` with a `composer.json`:

```
app/
  mymodule/
    composer.json    # Required: name, autoload
    module.php       # Optional: enabled, bindings
    src/
      MyService.php
```

Modules are discovered automatically. Use `module.php` for bindings:

```php title="module.php"
return [
    'enabled' => true,
    'bindings' => [
        PaymentInterface::class => StripePayment::class,
    ],
];
```

The `boot` callback runs after all module bindings are registered. Parameters are auto-injected from the container — type-hint any registered dependency:

```php title="module.php"
return [
    'bindings' => [
        PaymentInterface::class => StripePayment::class,
    ],
    'boot' => function (ErrorHandlerInterface $handler): void {
        $handler->register();
    },
];
```

### Application Environment

`AppEnvironment` is the single answer to "which environment is this application running in?". `Application` registers one shared instance in the container at boot, so any class or boot callback can type-hint it:

```php
use Marko\Core\Environment\AppEnvironment;

class ReportMailer
{
    public function __construct(
        private AppEnvironment $appEnvironment,
    ) {}

    public function recipients(): array
    {
        return $this->appEnvironment->isProduction()
            ? ['finance@example.com']
            : ['dev@example.com'];
    }
}
```

It reads `MARKO_ENV` first, then `APP_ENV`, from `$_ENV` with a `getenv()` fallback, so it works whether or not [marko/env](/docs/packages/env/) is installed and regardless of PHP's `variables_order` setting. Values are compared case-insensitively:

| Method | Returns `true` for |
|---|---|
| `isProduction()` | `production`, `prod`, or no value at all |
| `isDevelopment()` | `development`, `dev`, `local` |

Any other name (for example `staging` or `testing`) is neither production nor development. `name()` returns the trimmed, lowercased value, or `production` when neither variable is set (or both are empty). Defaulting to production means a deployment that forgets to set the environment fails safe instead of exposing development behavior.

### Environment-Specific Bindings

When different environments need different implementations (e.g., a mock service in development vs the real one in production), use the `boot` callback to conditionally override bindings:

```php title="module.php"
use Marko\Core\Container\Container;
use Marko\Core\Environment\AppEnvironment;

return [
    'bindings' => [
        // Default binding — used in all environments
        PaymentGatewayInterface::class => StripePaymentGateway::class,
    ],
    'boot' => function (Container $container, AppEnvironment $appEnvironment): void {
        if ($appEnvironment->isDevelopment()) {
            $container->bind(
                PaymentGatewayInterface::class,
                MockPaymentGateway::class,
            );
        }
    },
];
```

Since boot callbacks run after all static bindings are registered, `$container->bind()` in a boot callback overrides the static binding from the same module. The override is explicit and visible in the module's own `module.php`.

**Use boot callbacks when** you need a completely different implementation class per environment (mock vs real).

**Use config instead when** the difference is just values (API URLs, credentials, feature flags). Keep the same class everywhere and let [config](/docs/packages/config/) drive the behavior:

```php title="config/payments.php"
return [
    'gateway_url' => $_ENV['PAYMENT_GATEWAY_URL'] ?? 'https://sandbox.stripe.com',
    'dry_run' => (bool) ($_ENV['PAYMENT_DRY_RUN'] ?? true),
];
```

### Resetting Request-Scoped State in Long-Running Processes

PHP-FPM ends the process after every request, so any state a singleton accumulates disappears automatically. A long-running process --- a worker or event loop that reuses one PHP process across many requests --- has no such reset, so a singleton that caches per-request state (the current session, the authenticated user, a sticky database routing flag) leaks across requests once the process picks up a different user.

Implement `ResettableInterface` on any singleton that holds this kind of state:

```php
use Marko\Core\Contracts\ResettableInterface;

class RequestScopedCache implements ResettableInterface
{
    private array $entries = [];

    public function remember(string $key, mixed $value): void
    {
        $this->entries[$key] = $value;
    }

    public function reset(): void
    {
        $this->entries = [];
    }
}
```

`reset()` must be non-destructive --- it clears the instance's in-memory tracking, not anything persisted. Resetting a session service forgets which session the instance was serving; it does not delete the stored session.

A long-running process discovers what to reset via `Container::resolvedInstances()`, which returns only instances the container has already built --- never triggering resolution --- optionally filtered to those implementing an interface:

```php
use Marko\Core\Contracts\ResettableInterface;

foreach ($container->resolvedInstances(ResettableInterface::class) as $resettable) {
    $resettable->reset();
}
```

`Marko\Core\RequestStateResetter` wraps that loop: `new RequestStateResetter($container)->reset()` resets every resolved `ResettableInterface` instance in ascending binding-id order, and lets a failing `reset()` propagate. Pass instances to `reset(ResettableInterface ...$except)` to leave them alone (matched by identity); the testing database helpers use this to keep a test transaction open. The RoadRunner worker and the [marko/testing](/docs/packages/testing/) HTTP test client both call it before each request.

`resolvedInstances()` is declared on `ContainerInterface` and implemented by `Container`.

Current implementors: `Session` ([marko/session](/docs/packages/session/)), `SessionGuard` ([marko/authentication](/docs/packages/authentication/)), and `ReadWriteConnection` ([marko/database-readwrite](/docs/packages/database-readwrite/)).

### Registering Console Commands

Mark a class implementing `CommandInterface` with `#[Command]` and it is discovered automatically. Declare value-less boolean options in `flags` so they never swallow the positional argument that follows them:

```php title="app/billing/src/Command/RefundCommand.php"
use Marko\Core\Attributes\Command;
use Marko\Core\Command\CommandInterface;
use Marko\Core\Command\Input;
use Marko\Core\Command\Output;

#[Command(name: 'billing:refund', description: 'Refund an order', flags: ['force'])]
class RefundCommand implements CommandInterface
{
    public function execute(Input $input, Output $output): int
    {
        $orderId = $input->getArgument(0);           // positional --- options are never included
        $reason = $input->getOption('reason');       // --reason "late" or --reason=late
        $force = $input->hasOption('force');         // declared flag

        // ...

        return 0;
    }
}
```

`marko billing:refund --force 1001 --reason late` and `marko billing:refund 1001 --reason=late --force` are equivalent. Arguments and options can come in any order, `--` ends option parsing, and repeated options are read with `getOptionValues()`. See [marko/cli](/docs/packages/cli/#arguments-and-options) for the full option syntax.

### Discovery Cache

On every boot, Marko scans all module PHP files to discover `#[Preference]`, `#[Plugin]`, `#[Observer]`, and `#[Command]` attributes. In production this scan can be eliminated by compiling its results into a single PHP file --- the discovery cache.

**Routes are deliberately not part of the discovery cache.** `Application::initialize()` always runs route discovery live on every boot, in every environment --- so adding or changing a `#[Get]`, `#[Post]`, or any other route attribute takes effect on the next request with no rebuild, ever. The cache covers only the four attribute types listed above.

#### Compiling the cache

```bash
marko discovery:cache
```

Scans all modules, writes the compiled cache, and reports per-section counts:

```
Discovery cache compiled successfully.
Cache path: /var/www/html/storage/cache/discovery.php
preferences: 4
plugins: 12
observers: 7
commands: 9
```

#### Clearing the cache

```bash
marko discovery:clear
```

Deletes the compiled cache file. Idempotent --- safe to run when no cache exists.

#### How boot uses the cache

At boot, `Application::initialize()` reads these environment variables directly from `$_ENV`, falling back to `getenv()` (not via `marko/config`, so the gate works before any config package is loaded):

| Variable | Default | Description |
|---|---|---|
| `MARKO_ENV` / `APP_ENV` | `production` | Application environment, read through [`AppEnvironment`](#application-environment). `development`, `dev`, or `local` disable the cache. `MARKO_ENV` wins when both are set. |
| `DISCOVERY_CACHE_ENABLED` | `true` | Set to `0`, `false`, `no`, `off`, or empty to disable. |
| `DISCOVERY_CACHE_PATH` | `storage/cache/discovery.php` | Path to the cache file. Relative paths resolve from the project root; absolute paths are used as-is. |

The cache is used when **all three conditions** are true:

1. `DISCOVERY_CACHE_ENABLED` is truthy
2. `AppEnvironment::isDevelopment()` is false (the environment is not `development`, `dev`, or `local`)
3. The cache file exists at `DISCOVERY_CACHE_PATH`

If the cache file is **missing**, boot falls back to a normal full rescan --- no error.

If the cache file is **corrupt, malformed, or version-mismatched**, boot throws `DiscoveryCacheException` immediately. There is no silent fallback. Run `marko discovery:clear` then `marko discovery:cache` to rebuild.

In a **development** environment (`development`, `dev`, or `local` --- the skeleton ships `APP_ENV=local`) the cache is always bypassed, so adding or editing a `#[Plugin]`, `#[Observer]`, `#[Preference]`, or `#[Command]` takes effect on the next request without any manual step.

#### Configuration via `marko/config`

When `marko/config` is installed, the same keys are available as a config file:

```php title="config/discovery.php"
return [
    'enabled'    => true,   // mirrors DISCOVERY_CACHE_ENABLED
    'environment' => 'production', // mirrors APP_ENV
    'cache_path' => 'storage/cache/discovery.php', // mirrors DISCOVERY_CACHE_PATH
];
```

The core-owned `config/discovery.php` is shipped with `marko/core` and populates these values from `$_ENV` automatically. The boot gate reads `DiscoveryEnvironment` directly and does not depend on `marko/config`.

#### Deploy requirement

There is no file-modification-time invalidation. Whenever code changes (new modules, updated attributes), regenerate the cache as part of your deploy:

```bash
composer install --no-dev --optimize-autoloader
marko discovery:cache
```

Serving stale discovery results in missing preferences, plugins, observers, or commands until the cache is recompiled.

Recompiling on every deploy also covers framework upgrades that change the cache format. A cache written by an older version fails boot with a version-mismatch `DiscoveryCacheException` rather than loading incomplete data (for example, command `flags` were added in cache version 2).

#### Not the same as the code index

Marko has **two separate caches** that are easy to confuse --- different files, different commands, different consumers:

| Cache | File | Built by | Consumed by | Rebuild when |
|---|---|---|---|---|
| **Discovery cache** | `storage/cache/discovery.php` | `marko discovery:cache` | the **running app** at boot | deploying to a non-`development` environment after plugins, observers, preferences, or commands changed |
| **Code index** | `.marko/index.cache` | `marko indexer:rebuild` | **MCP / LSP tooling** ([`marko/codeindexer`](/docs/packages/codeindexer/)) | the AI tools show stale or missing symbols |

The discovery cache makes the **app** boot faster in production. The code index lets **AI tooling** answer questions about your code. Rebuilding one has no effect on the other. Neither is required for routes or newly-added modules to work at runtime --- see [the routing note above](#discovery-cache) and the codeindexer page.

### Throwing Rich Exceptions

Include context and fix suggestions:

```php
use Marko\Core\Exceptions\MarkoException;

throw new MarkoException(
    message: 'Payment failed',
    context: 'Processing order #12345',
    suggestion: 'Check that the API key is configured in .env',
);
```

## API Reference

### Attributes

```php
#[Preference(replaces: ClassName::class)]      // Replace a class globally
#[Plugin(target: ClassName::class)]            // Mark class as plugin
#[Before]                                       // Run before target method
#[After]                                        // Run after target method
#[Observer(event: EventClass::class)]           // React to events (synchronous)
#[Observer(event: EventClass::class, async: true)] // Queue it (needs marko/queue; throws EventException without it)
#[Command(name: 'cmd:name', description: '', aliases: [], flags: [])] // Register CLI command; flags never take a value
```

### Container

```php
interface ContainerInterface extends PsrContainerInterface
{
    public function get(string $id): mixed;
    public function has(string $id): bool;
    public function singleton(string $id): void;
    public function instance(string $id, object $instance): void;
    public function call(Closure $callable): mixed;
}
```

`has()` returns `true` for existing classes, interface bindings and instances registered with `instance()`, so it works for an interface that only has a pre-built instance and no binding.

The concrete `Container` class additionally provides `resolvedInstances(?string $interface = null): array` --- not part of `ContainerInterface`. It returns only instances already built, optionally filtered to those implementing `$interface`, and never triggers resolution as a side effect. See [Resetting Request-Scoped State](#resetting-request-scoped-state-in-long-running-processes) above.

### AppEnvironment

```php
use Marko\Core\Environment\AppEnvironment;

class AppEnvironment
{
    public function __construct(?array $variables = null);

    public function name(): string;
    public function isProduction(): bool;
    public function isDevelopment(): bool;
}
```

Registered as a shared container instance by `Application`. Pass `$variables` (for example `['APP_ENV' => 'local']`) to read from that array only, instead of the real environment --- useful in tests. See [Application Environment](#application-environment) above.

### Contracts

```php
interface ResettableInterface
{
    public function reset(): void;
}
```

Implemented by services that hold request-scoped state which must be cleared between requests in a long-running process. See [Resetting Request-Scoped State](#resetting-request-scoped-state-in-long-running-processes) above.

### Events

```php
interface EventDispatcherInterface
{
    public function dispatch(Event $event): void;
}
```

### MarkoException

```php
class MarkoException extends Exception
{
    public function __construct(
        string $message,
        string $context = '',
        string $suggestion = '',
    );

    public function getContext(): string;
    public function getSuggestion(): string;
}
```

### CircularDependencyException

```php
use Marko\Core\Exceptions\CircularDependencyException;
```

Thrown by the container when a mutual constructor dependency cycle is detected (e.g. class A requires class B which requires class A). Rather than exhausting the call stack, the container detects the cycle and throws immediately with a human-readable chain (`A -> B -> A`) and a suggestion to remove the circular reference.

Implements `Psr\Container\ContainerExceptionInterface`.

### DiscoveryCacheException

```php
use Marko\Core\Exceptions\DiscoveryCacheException;
```

Thrown by `Application::initialize()` when the discovery cache file is corrupt, structurally invalid, or carries a version number that does not match the running core version. There is no silent fallback --- a bad cache is a loud error.

Named constructors:

| Method | When thrown |
|---|---|
| `DiscoveryCacheException::unreadable($path)` | Cache file exists but cannot be read |
| `DiscoveryCacheException::malformed($path, $reason)` | Cache file structure is invalid or missing required keys |
| `DiscoveryCacheException::versionMismatch($path, $found, $expected)` | Cache was compiled by a different core version |
| `DiscoveryCacheException::notWritable($path)` | Cache directory or file is not writable (thrown by `discovery:cache`) |

Fix in all cases: `marko discovery:clear && marko discovery:cache`.
