---
title: marko/authentication-token
description: Stateless API token authentication --- issue personal access tokens with scoped abilities for mobile apps, SPAs, and third-party integrations.
---

Stateless API token authentication --- issue personal access tokens with scoped abilities for mobile apps, SPAs, and third-party integrations. This package adds personal access token authentication to the Marko Framework. Each token is stored as a SHA-256 hash; the plain-text value is only available once at creation time. Tokens can be scoped to a list of abilities so you can issue limited-permission tokens for specific integrations. `TokenGuard` serves the `token` guard driver: once the package is installed, every guard configured with `'driver' => 'token'` authenticates API requests from the `Authorization: Bearer` header.

## Installation

```bash
composer require marko/authentication-token
```

## Configuration

Override defaults in `config/authentication-token.php`:

```php title="config/authentication-token.php"
return [
    // Days before a token expires. null = never expires.
    'token_expiration_days' => 365,
];
```

## Usage

### Creating a Token

Inject `TokenManager` and call `createToken()`. Capture `plainTextToken` immediately --- it is never retrievable again.

```php title="ApiTokenController.php"
use DateTimeImmutable;
use Marko\AuthenticationToken\Service\TokenManager;
use Marko\Authentication\AuthenticatableInterface;

class ApiTokenController
{
    public function __construct(
        private TokenManager $tokenManager,
    ) {}

    public function store(
        AuthenticatableInterface $user,
    ): Response {
        $newToken = $this->tokenManager->createToken(
            user: $user,
            name: 'mobile-app',
            abilities: ['posts:read', 'posts:write'],
        );

        // plainTextToken is ONLY available here — store it immediately
        return new Response([
            'token' => $newToken->plainTextToken,
        ]);
    }

    public function storeWithExpiry(
        AuthenticatableInterface $user,
    ): Response {
        $newToken = $this->tokenManager->createToken(
            user: $user,
            name: 'short-lived',
            abilities: ['posts:read'],
            expiresAt: new DateTimeImmutable('+30 days'),
        );

        return new Response([
            'token' => $newToken->plainTextToken,
        ]);
    }
}
```

The `expiresAt` parameter is optional (`null` by default). A `null` expiry means the token never expires. When a non-null expiry is set and the current time is past it, `TokenGuard` rejects the token and returns `null` from `user()`.

> **Security note:** The plain-text token is available only on `NewAccessToken::$plainTextToken` at creation time. The database stores only a SHA-256 hash. If you lose the plain text, you must revoke and re-issue.

### Wiring the Token Guard

Installing the package registers the `token` guard driver with `AuthManager` (through `GuardDriverRegistry`, from the package's `module.php` `boot` callback). Any guard configured with `'driver' => 'token'` resolves to `Marko\AuthenticationToken\Guard\TokenGuard`, so `AuthManager::guard()`, `GuardInterface`, `AuthMiddleware`, `#[Can]` and the `Gate` all use it, with expiry enforced. Point a guard at the driver in `config/authentication.php`:

```php title="config/authentication.php"
return [
    'default' => [
        'guard' => 'api',
        'provider' => 'users',
    ],
    'guards' => [
        'session' => ['driver' => 'session', 'provider' => 'users'],
        'api' => ['driver' => 'token', 'provider' => 'users'],
    ],
    // ...
];
```

Bind your `TokenRepositoryInterface` implementation (backed by the `personal_access_tokens` table) in your app's `module.php`. The guard resolves it the first time a token guard is built, so commands and requests that never touch a token guard do not need it.

The package also registers `TokenRequestMiddleware` as global middleware. It hands each inbound request to the guard (through the `CurrentRequest` singleton) and clears it when the request ends, so a long-running worker never reuses an earlier request's token. It is sequenced before `marko/authorization`, so `#[Can]` sees the token. Outside an HTTP request (CLI, queue jobs), the token guard has no request and treats the caller as a guest.

### Bearer Token Authentication Flow

Clients send the token in the `Authorization` header on every request:

```
Authorization: Bearer <plain-text-token>
```

`TokenGuard` hashes the token, looks it up in `personal_access_tokens`, rejects it once `expires_at` has passed, and resolves the user through the configured user provider. Protect routes with `AuthMiddleware` on the token guard:

```php title="ApiController.php"
use Marko\Authentication\AuthManager;
use Marko\Authentication\Middleware\AuthMiddleware;
use Marko\Routing\Attributes\Get;
use Marko\Routing\Attributes\Middleware;

class ApiController
{
    public function __construct(
        private AuthManager $authManager,
    ) {}

    #[Get('/api/me')]
    #[Middleware(AuthMiddleware::class)] // default guard 'api' uses the token driver
    public function me(): Response
    {
        $user = $this->authManager->guard('api')->user(); // resolved from the Bearer token

        return new Response("Hello, user {$user->getAuthIdentifier()}");
    }
}
```

A missing, unknown, revoked or expired token gets a `401` with `WWW-Authenticate: Bearer` from `AuthMiddleware`, never a login redirect, because the token guard implements `StatelessGuardInterface`. `#[Can]` answers the same requests with a `401` too.

### Stateless Methods

The token guard keeps no login state. `attempt()`, `login()`, `loginById()` and `logout()` throw a `StatelessGuardException` (an `AuthException`) that points you to `TokenManager`: issue a token with `createToken()` and end access with `revokeToken()` or `revokeAllTokens()`.

### Checking Token Abilities

After authentication, check whether the resolved token has a specific ability. `hasAbility()` lives on `TokenGuard`, so narrow the guard first:

```php
use Marko\AuthenticationToken\Guard\TokenGuard;

public function update(
    Request $request,
): Response {
    $guard = $this->authManager->guard('api');

    if (!$guard instanceof TokenGuard || !$guard->hasAbility('posts:write')) {
        return new Response('Forbidden', 403);
    }

    // proceed with update
}
```

Abilities are stored as a JSON array on the token record. Pass an empty `abilities` array to `createToken()` for a token with no scoping (full access).

### Revoking Tokens

Revoke a single token by its database ID:

```php
$this->tokenManager->revokeToken(
    tokenId: $token->id,
);
```

Revoke all tokens belonging to a user:

```php
$this->tokenManager->revokeAllTokens(
    user: $user,
);
```

## Events

The package dispatches token lifecycle [events](/docs/packages/events/) through core's `EventDispatcherInterface`. It never fires the session `LoginEvent`/`LogoutEvent`, since a token guard has no login. No event carries the token value, plain-text or hashed, so every event is safe to log.

| Event | Dispatched by | Properties |
|---|---|---|
| `TokenCreatedEvent` | `TokenManager::createToken()` | `user`, `tokenId`, `name`, `abilities`, `expiresAt` (`?DateTimeImmutable`) |
| `TokenRevokedEvent` | `TokenManager::revokeToken()` | `tokenId` |
| `AllTokensRevokedEvent` | `TokenManager::revokeAllTokens()` | `user` |
| `TokenAuthenticationFailedEvent` | `TokenGuard`, when a request presents an unknown, revoked or expired token | `guard`, `reason` (`TokenFailureReason::Invalid` or `::Expired`), `tokenId` (expired tokens only), `ipAddress` |

`TokenAuthenticationFailedEvent` fires at most once per request and guard. A request without a bearer token and a successful authentication fire nothing: authenticating every API request is too noisy to be an event.

All events live in `Marko\AuthenticationToken\Event`.

```php title="NotifyNewTokenObserver.php"
use Marko\AuthenticationToken\Event\TokenCreatedEvent;
use Marko\Core\Attributes\Observer;

#[Observer(TokenCreatedEvent::class)]
class NotifyNewTokenObserver
{
    public function handle(
        TokenCreatedEvent $event,
    ): void {
        // Email $event->user: "A new API token '$event->name' was created"
    }
}
```

```php title="ThrottleBadTokensObserver.php"
use Marko\AuthenticationToken\Event\TokenAuthenticationFailedEvent;
use Marko\Core\Attributes\Observer;

#[Observer(TokenAuthenticationFailedEvent::class)]
class ThrottleBadTokensObserver
{
    public function handle(
        TokenAuthenticationFailedEvent $event,
    ): void {
        // Count failures per $event->ipAddress and block repeat offenders
    }
}
```

## Database

The package uses the `personal_access_tokens` table. Columns:

| Column | Type | Description |
|---|---|---|
| `id` | int | Primary key |
| `tokenable_type` | string | User class name |
| `tokenable_id` | int | User identifier |
| `name` | string | Human-readable token name |
| `token_hash` | string(64) | SHA-256 hash of the plain-text token |
| `abilities` | text | JSON-encoded array of ability strings |
| `last_used_at` | datetime | Last usage timestamp |
| `expires_at` | datetime | Expiry timestamp (null = never) |
| `created_at` | datetime | Creation timestamp |

## API Reference

### TokenManager

```php
use DateTimeInterface;
use Marko\Authentication\AuthenticatableInterface;
use Marko\AuthenticationToken\Contracts\NewAccessToken;

public function createToken(AuthenticatableInterface $user, string $name, array $abilities = [], ?DateTimeInterface $expiresAt = null): NewAccessToken;
public function revokeToken(int $tokenId): void;
public function revokeAllTokens(AuthenticatableInterface $user): void;
```

### TokenGuard

Implements `StatelessGuardInterface`:

```php
public function __construct(TokenRepositoryInterface $repository, CurrentRequest $currentRequest, ClockInterface $clock, UserProviderInterface $provider, string $name = 'token', ?EventDispatcherInterface $eventDispatcher = null);
public function check(): bool;
public function guest(): bool;
public function user(): ?AuthenticatableInterface;
public function id(): int|string|null;
public function hasAbility(string $ability): bool;
public function extractToken(): ?string;
public function getName(): string;
public function getChallenge(): string; // 'Bearer'
public function attempt(array $credentials): bool; // throws StatelessGuardException
public function login(AuthenticatableInterface $user): void; // throws StatelessGuardException
public function loginById(int|string $id): ?AuthenticatableInterface; // throws StatelessGuardException
public function logout(): void; // throws StatelessGuardException
```

### TokenGuardFactory

Builds the guard for the `token` driver:

```php
public function create(string $name, UserProviderInterface $provider): TokenGuard;
```

### CurrentRequest

Singleton holding the request being handled, set by `TokenRequestMiddleware`. Implements `ResettableInterface`:

```php
public function set(Request $request): void;
public function get(): ?Request;
public function reset(): void;
```

### NewAccessToken

```php
public readonly PersonalAccessToken $accessToken;
public readonly string $plainTextToken; // available once at creation only
```

### TokenRepositoryInterface

```php
public function find(int $id): ?PersonalAccessToken;
public function findByToken(string $tokenHash): ?PersonalAccessToken;
public function create(PersonalAccessToken $token): PersonalAccessToken;
public function revoke(int $id): void;
public function revokeAllForUser(string $type, int|string $id): void;
```

### HasApiTokensInterface

```php
public function getTokens(): array;
public function createToken(string $name, array $abilities = []): NewAccessToken;
```

### Exceptions

```php
InvalidTokenException::forToken(): self;
ExpiredTokenException::forToken(?int $tokenId, DateTimeInterface $expiredAt): self;
StatelessGuardException::forMethod(string $guard, string $method): self;
```

`InvalidTokenException` and `ExpiredTokenException` extend `TokenException`, which exposes `getContext(): string` and `getSuggestion(): string` for detailed error messages. Neither takes the token value, so it never reaches logs or error pages. `StatelessGuardException` extends `Marko\Authentication\Exceptions\AuthException`.

## Related Packages

- [marko/authentication](/docs/packages/authentication/) --- Core authentication framework with guards, user providers, middleware, and events.
