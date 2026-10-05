---
title: marko/testing
description: Reusable fakes with built-in assertions that eliminate test boilerplate.
---

Testing utilities for Marko — reusable fakes with built-in assertions that eliminate test boilerplate. This package provides in-memory fakes for the core Marko contracts: events, mail, queues, sessions, cookies, logging, config, authentication, guards, HTTP clients, and the clock. Each fake records interactions and exposes assertion methods so your tests stay focused on behavior rather than mock setup. Pest expectation extensions (`toHaveDispatched`, `toHaveSent`, `toHavePushed`, `toHaveLogged`, `toHaveAttempted`, `toBeAuthenticated`) are included for fluent assertions.

Available fakes: `FakeEventDispatcher`, `FakeMailer`, `FakeQueue`, `FakeSession`, `FakeCookieJar`, `FakeLogger`, `FakeConfigRepository`, `FakeAuthenticatable`, `FakeUserProvider`, `FakeGuard`, `FakeHttpClient`, `FakeClock`.

## Installation

```bash
composer require marko/testing --dev
```

## Usage

### FakeEventDispatcher

```php
use Marko\Testing\Fake\FakeEventDispatcher;

$dispatcher = new FakeEventDispatcher();
$dispatcher->dispatch(new OrderPlaced($order));

$dispatcher->assertDispatched(OrderPlaced::class);
$dispatcher->assertDispatchedCount(OrderPlaced::class, 1);
$dispatcher->assertNotDispatched(OrderShipped::class);
```

### FakeMailer

```php
use Marko\Testing\Fake\FakeMailer;

$mailer = new FakeMailer();
$mailer->send($message);

$mailer->assertSent();
$mailer->assertSent(fn (Message $m) => $m->subject === 'Welcome');
$mailer->assertSentCount(1);
$mailer->assertNothingSent();
```

### FakeQueue

```php
use Marko\Testing\Fake\FakeQueue;

$queue = new FakeQueue();
$queue->push(new SendEmailJob($user));

$queue->assertPushed(SendEmailJob::class);
$queue->assertPushed(SendEmailJob::class, fn ($job) => $job->userId === $user->id);
$queue->assertPushedCount(1);
$queue->assertNothingPushed();
$queue->assertNotPushed(ProcessPaymentJob::class);
```

### FakeSession

```php
use Marko\Testing\Fake\FakeSession;

$session = new FakeSession();
$session->start();
$session->set('user_id', 42);

expect($session->started)->toBeTrue()
    ->and($session->get('user_id'))->toBe(42)
    ->and($session->has('user_id'))->toBeTrue();

$session->destroy();
expect($session->destroyed)->toBeTrue();
```

`has()` returns `true` when a key is stored, even if its value is `null`. This matches the behavior of the production `Session` implementation.

### FakeCookieJar

```php
use Marko\Testing\Fake\FakeCookieJar;

$cookies = new FakeCookieJar();
$cookies->set('remember_me', 'token-value', minutes: 60);

expect($cookies->get('remember_me'))->toBe('token-value');

$cookies->delete('remember_me');
expect($cookies->get('remember_me'))->toBeNull();
```

### FakeLogger

```php
use Marko\Testing\Fake\FakeLogger;
use Marko\Log\LogLevel;

$logger = new FakeLogger();
$logger->info('User logged in', ['user_id' => 1]);
$logger->error('Payment failed');

$logger->assertLogged('User logged in');
$logger->assertLogged('User logged in', LogLevel::Info);
$logger->assertNothingLogged(); // Would fail here
$logger->clear();
$logger->assertNothingLogged(); // Passes after clear
```

### FakeConfigRepository

```php
use Marko\Testing\Fake\FakeConfigRepository;

$config = new FakeConfigRepository([
    'app.name' => 'Marko',
    'mail.driver' => 'smtp',
    'default.cache.ttl' => 3600,
    'scopes.store_1.cache.ttl' => 7200,
]);

expect($config->getString('app.name'))->toBe('Marko')
    ->and($config->getInt('default.cache.ttl'))->toBe(3600);

$config->set('app.debug', true);
expect($config->getBool('app.debug'))->toBeTrue();
```

### FakeGuard

```php
use Marko\Testing\Fake\FakeAuthenticatable;
use Marko\Testing\Fake\FakeGuard;

$guard = new FakeGuard(name: 'web', attemptResult: true);

// Set current user directly
$user = new FakeAuthenticatable(id: 1);
$guard->setUser($user);

expect($guard->check())->toBeTrue()
    ->and($guard->id())->toBe(1);

// Simulate a login attempt
$guard->setUser(null);
$guard->attempt(['email' => 'user@example.com', 'password' => 'secret']);

$guard->assertAttempted();
$guard->assertAttempted(fn (array $creds) => $creds['email'] === 'user@example.com');

// Simulate logout
$guard->logout();
$guard->assertLoggedOut();
$guard->assertGuest();
```

### FakeClock

`FakeClock` implements the PSR-20 `Psr\Clock\ClockInterface`, so it drops into anything that takes the injected clock from [`marko/clock`](/docs/packages/clock/). It is frozen: time never moves on its own, so expiry, TTL and freshness checks can be tested at exact boundaries without `sleep()` or loose time ranges.

```php
use Marko\Testing\Fake\FakeClock;

$clock = new FakeClock('2026-01-01 12:00:00 UTC');
$guard = new TokenGuard($repository, $request, $clock);

$clock->now();                         // always 2026-01-01 12:00:00
$clock->travel('+59 minutes');         // relative move
$clock->travelTo('2026-01-02 00:00');  // absolute jump
$clock->setNow(new DateTimeImmutable('2026-06-01 09:00:00 UTC'));

$clock->assertNowIs('2026-06-01 09:00:00 UTC');
```

`travel()` accepts any `DateTimeImmutable::modify()` string; a malformed modifier throws `DateMalformedStringException`. With no argument, the fake is frozen at the moment it was created.

### FakeAuthenticatable and FakeUserProvider

```php
use Marko\Testing\Fake\FakeAuthenticatable;
use Marko\Testing\Fake\FakeUserProvider;

$user = new FakeAuthenticatable(id: 1, password: 'hashed');
$provider = new FakeUserProvider(users: [1 => $user]);

$found = $provider->retrieveById(1);
expect($found)->toBe($user);

$valid = $provider->validateCredentials($user, ['password' => 'secret']);
expect($valid)->toBeTrue(); // Default validator always returns true

$provider->updateRememberToken($user, 'new-token');
expect($provider->lastRememberTokenUpdate['token'])->toBe('new-token');
```

### FakeHttpClient

`FakeHttpClient` implements `HttpClientInterface` from [`marko/http`](/docs/packages/http/), so any service that takes the interface can be tested without a live server or Guzzle internals.

```php
use Marko\Http\Exceptions\ConnectionException;
use Marko\Http\HttpResponse;
use Marko\Testing\Fake\FakeHttpClient;
use Marko\Testing\Fake\Http\RecordedRequest;

$http = new FakeHttpClient();

// Exact URL, or * as a wildcard
$http->stub('https://api.example.com/orders/*', new HttpResponse(200, '{"id":1}', ['Content-Type' => 'application/json']));
$http->stub('https://api.example.com/fail', new HttpResponse(500, 'boom'));

// Simulate a network failure
$http->stub('https://api.example.com/slow', new ConnectionException('timeout'));

// Sequential responses for requests that match no stub, whatever their URL
$http->queue(new HttpResponse(201, ''), new HttpResponse(200, ''));

$service = new OrderSync($http);
$service->push($order);

$http->assertSent(fn (RecordedRequest $r) => $r->method === 'POST' && $r->json()['id'] === 1);
$http->assertSentCount(1);
$http->assertNotSent(fn (RecordedRequest $r) => str_contains($r->url, '/fail'));
```

Each request is resolved in this order: the first registered stub whose pattern matches the URL, then the next queued response. A request that matches neither is a **stray request** and throws `AssertionFailedException`, so a test never silently talks to an unexpected endpoint. Call `$http->preventStrayRequests(false)` to answer stray requests with an empty `200` response instead.

The fake applies the same rules as a real driver:

- Options are validated with `RequestOptions::validate()`, so an unknown key such as `form_param` (or the Guzzle-only `guzzle` key) throws `InvalidRequestOptionException`.
- A stubbed 4xx/5xx response throws `HttpException` with the response attached, unless the request passes `'http_errors' => false`, in which case the `HttpResponse` is returned.

Every sent request is recorded as a readonly `RecordedRequest` in `$http->requests`, with `method` (uppercased), `url` and `options`, plus helpers:

```php
$request = $http->requests[0];

$request->header('authorization'); // case-insensitive, null when absent
$request->json();                  // the 'json' option
$request->body();                  // raw 'body', encoded 'json', or encoded 'form_params'
```

### KnownDriversValidator

`KnownDriversValidator` is a static utility for package authors to assert that a package's `known-drivers.php` file is well-formed and stays in sync with the skeleton's `suggest` block. See [Known Drivers](/concepts/known-drivers/) for the file format and description-string conventions.

```php
use Marko\Testing\KnownDrivers\KnownDriversValidator;

// Assert every key in known-drivers.php follows the 'marko/*' prefix pattern
KnownDriversValidator::assertDocsUrlsResolveToValidPattern(
    __DIR__ . '/../known-drivers.php',
);

// Assert skeleton composer.json suggest block contains every entry from known-drivers.php
KnownDriversValidator::assertSkeletonSuggestContainsAll(
    __DIR__ . '/../known-drivers.php',
    __DIR__ . '/../../skeleton/composer.json',
);
```

## Pest Expectations

The expectations register automatically. `marko/testing` declares a Pest plugin (`Marko\Testing\Pest\ExpectationsPlugin`) under `extra.pest.plugins` in its `composer.json`, and Pest boots it once `expect()` exists, including in `--parallel` workers. There is nothing to add to `Pest.php`.

The expectations need Pest 4 (`composer require --dev pestphp/pest`). The fakes and their `assert*()` methods work with plain PHPUnit.

:::note
If an expectation such as `toHavePushed` is reported as an undefined method, Pest's plugin list is stale. Run `composer dump-autoload` to regenerate `vendor/pest-plugins.json`, and make sure `pestphp/pest-plugin` is allowed under `config.allow-plugins`.
:::

Use them in tests:

```php
expect($dispatcher)->toHaveDispatched(OrderPlaced::class);
expect($mailer)->toHaveSent();
expect($mailer)->toHaveSent(fn (Message $m) => $m->to === 'user@example.com');
expect($queue)->toHavePushed(SendEmailJob::class);
expect($logger)->toHaveLogged('Payment failed');
expect($logger)->toHaveLogged('Payment failed', LogLevel::Error);
expect($http)->toHaveSentRequest();
expect($http)->toHaveSentRequest(fn (RecordedRequest $r) => $r->url === 'https://api.example.com/orders');
```

## API Reference

### FakeEventDispatcher

```php
public function dispatch(Event $event): void;
public function dispatched(string $eventClass): array;
public function assertDispatched(string $eventClass): void;
public function assertNotDispatched(string $eventClass): void;
public function assertDispatchedCount(string $eventClass, int $expected): void;
public function clear(): void;
```

### FakeMailer

```php
public function send(Message $message): bool;
public function sendRaw(string $to, string $raw): bool;
public function assertSent(?callable $callback = null): void;
public function assertNothingSent(): void;
public function assertSentCount(int $expected): void;
public function clear(): void;
```

### FakeQueue

```php
public function push(JobInterface $job, ?string $queue = null): string;
public function later(int $delay, JobInterface $job, ?string $queue = null): string;
public function pop(?string $queue = null): ?JobInterface;
public function size(?string $queue = null): int;
public function clear(?string $queue = null): int;
public function delete(string $jobId): bool;
public function release(string $jobId, int $delay = 0): bool;
public function assertPushed(string $jobClass, ?callable $callback = null): void;
public function assertNotPushed(string $jobClass): void;
public function assertPushedCount(int $expected): void;
public function assertNothingPushed(): void;
```

### FakeSession

```php
public function start(): void;
public function get(string $key, mixed $default = null): mixed;
public function set(string $key, mixed $value): void;
public function has(string $key): bool;
public function remove(string $key): void;
public function clear(): void;
public function all(): array;
public function regenerate(bool $deleteOldSession = true): void;
public function destroy(): void;
public function getId(): string;
public function setId(string $id): void;
public function flash(): FlashBag;
public function save(): void;
```

### FakeCookieJar

```php
public function get(string $name): ?string;
public function set(string $name, string $value, int $minutes = 0): void;
public function delete(string $name): void;
```

### FakeLogger

```php
public function emergency(string $message, array $context = []): void;
public function alert(string $message, array $context = []): void;
public function critical(string $message, array $context = []): void;
public function error(string $message, array $context = []): void;
public function warning(string $message, array $context = []): void;
public function notice(string $message, array $context = []): void;
public function info(string $message, array $context = []): void;
public function debug(string $message, array $context = []): void;
public function log(LogLevel $level, string $message, array $context = []): void;
public function entriesForLevel(LogLevel $level): array;
public function assertLogged(string $message, ?LogLevel $level = null): void;
public function assertNothingLogged(): void;
public function clear(): void;
```

### FakeConfigRepository

```php
public function __construct(array $config = [], ?string $defaultScope = null);
public function get(string $key, ?string $scope = null): mixed;
public function has(string $key, ?string $scope = null): bool;
public function getString(string $key, ?string $scope = null): string;
public function getInt(string $key, ?string $scope = null): int;
public function getBool(string $key, ?string $scope = null): bool;
public function getFloat(string $key, ?string $scope = null): float;
public function getArray(string $key, ?string $scope = null): array;
public function all(?string $scope = null): array;
public function withScope(string $scope): ConfigRepositoryInterface;
public function set(string $key, mixed $value): void;
```

### FakeAuthenticatable

```php
public function __construct(int|string $id = 1, string $password = 'hashed-password', ?string $rememberToken = null, string $identifierName = 'id', string $rememberTokenName = 'remember_token');
public function getAuthIdentifier(): int|string;
public function getAuthIdentifierName(): string;
public function getAuthPassword(): string;
public function getRememberToken(): ?string;
public function setRememberToken(?string $token): void;
public function getRememberTokenName(): string;
```

### FakeUserProvider

```php
public function __construct(array $users = [], ?callable $credentialValidator = null);
public function retrieveById(int|string $identifier): ?AuthenticatableInterface;
public function retrieveByCredentials(array $credentials): ?AuthenticatableInterface;
public function validateCredentials(AuthenticatableInterface $user, array $credentials): bool;
public function retrieveByRememberToken(int|string $identifier, string $token): ?AuthenticatableInterface;
public function updateRememberToken(AuthenticatableInterface $user, ?string $token): void;
```

### FakeGuard

```php
public function __construct(string $name = 'test', bool $attemptResult = true);
public function check(): bool;
public function guest(): bool;
public function user(): ?AuthenticatableInterface;
public function id(): int|string|null;
public function attempt(array $credentials): bool;
public function login(AuthenticatableInterface $user): void;
public function loginById(int|string $id): ?AuthenticatableInterface;
public function logout(): void;
public function getName(): string;
public function setUser(?AuthenticatableInterface $user): void;
public function setAttemptResult(bool $result): void;
public function assertAuthenticated(): void;
public function assertGuest(): void;
public function assertAttempted(?callable $callback = null): void;
public function assertNotAttempted(): void;
public function assertLoggedOut(): void;
public function clear(): void;
```

### FakeHttpClient

```php
public array $requests; // array<RecordedRequest>, read-only from outside
public function stub(string $urlPattern, HttpResponse|HttpException $response): self;
public function queue(HttpResponse|HttpException ...$responses): self;
public function preventStrayRequests(bool $prevent = true): self;
public function request(string $method, string $url, array $options = []): HttpResponse;
public function get(string $url, array $options = []): HttpResponse;
public function post(string $url, array $options = []): HttpResponse;
public function put(string $url, array $options = []): HttpResponse;
public function patch(string $url, array $options = []): HttpResponse;
public function delete(string $url, array $options = []): HttpResponse;
public function assertSent(?callable $callback = null): void;
public function assertNotSent(callable $callback): void;
public function assertSentCount(int $expected): void;
public function assertNothingSent(): void;
public function clear(): void;
```

### RecordedRequest

```php
public function __construct(string $method, string $url, array $options = []);
public function header(string $name): ?string;
public function json(): mixed;
public function body(): string;
```

### FakeClock

```php
public function __construct(DateTimeImmutable|string $now = 'now');
public function now(): DateTimeImmutable;
public function setNow(DateTimeImmutable|string $now): void;
public function travel(string $modifier): void;
public function travelTo(DateTimeImmutable|string $now): void;
public function assertNowIs(DateTimeImmutable|string $expected): void;
```

### KnownDriversValidator

```php
public static function assertSkeletonSuggestContainsAll(string $knownDriversPath, string $skeletonComposerPath): void;
public static function assertDocsUrlsResolveToValidPattern(string $knownDriversPath): void;
```

### AssertionFailedException

```php
public static function expectedDispatched(string $eventClass): self;
public static function unexpectedDispatched(string $eventClass): self;
public static function expectedCount(string $type, int $expected, int $actual): self;
public static function expectedContains(string $type, string $needle): self;
public static function unexpectedContains(string $type, string $needle): self;
public static function expectedEmpty(string $type): self;
public static function unexpectedEmpty(string $type): self;
public static function strayRequest(string $method, string $url): self;
```
