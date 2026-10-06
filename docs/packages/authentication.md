---
title: marko/authentication
description: Session and token-based authentication — guards protect routes, events track activity, middleware controls access.
---

Session and token-based authentication — guards protect routes, events track activity, middleware controls access. The auth package provides flexible authentication with two built-in guards: `SessionGuard` for web applications and `TokenGuard` for APIs. Configure multiple guards, implement custom user providers, and react to authentication events via observers.

## Installation

```bash
composer require marko/authentication
```

## Configuration

The package reads configuration from `config/authentication.php`:

```php title="config/authentication.php"
return [
    'default' => [
        'guard' => 'session',
        'provider' => 'users',
    ],

    'guards' => [
        'session' => [
            'driver' => 'session',
            'provider' => 'users',
        ],
        'token' => [
            'driver' => 'token',
            'provider' => 'users',
        ],
    ],

    'providers' => [
        'users' => [
            'driver' => 'database',
            'table' => 'users',
        ],
    ],

    'password' => [
        'driver' => 'bcrypt',
        'bcrypt' => [
            'cost' => 12,
        ],
    ],

    'remember' => [
        'lifetime' => 43200, // minutes (30 days)
        'cookie' => [
            'prefix' => 'remember_',
            'path' => '/',
            'domain' => '',
            'secure' => null, // null follows session.cookie.secure
            'http_only' => true,
            'same_site' => 'Lax',
        ],
    ],
];
```

See [Remember Me](#remember-me) for what each `remember` option controls.

## Usage

### Checking Authentication

Inject `AuthManager` to check if a user is authenticated:

```php title="DashboardController.php"
use Marko\Authentication\AuthManager;

class DashboardController
{
    public function __construct(
        private AuthManager $authManager,
    ) {}

    public function index(): Response
    {
        if ($this->authManager->check()) {
            $user = $this->authManager->user();
            return new Response("Welcome, user {$this->authManager->id()}");
        }

        return Response::redirect('/login');
    }
}
```

### Logging In

Use `attempt()` to authenticate with credentials:

```php
public function login(
    Request $request,
): Response {
    $credentials = [
        'email' => $request->get('email'),
        'password' => $request->get('password'),
    ];

    if ($this->authManager->attempt($credentials)) {
        return Response::redirect('/dashboard');
    }

    return new Response('Invalid credentials', 401);
}
```

### Logging Out

```php
public function logout(): Response
{
    $this->authManager->logout();

    return Response::redirect('/');
}
```

### Making Users Authenticatable

Your user model must implement `AuthenticatableInterface`:

```php title="User.php"
use Marko\Authentication\AuthenticatableInterface;

class User implements AuthenticatableInterface
{
    public function __construct(
        private int $id,
        private string $email,
        private string $password,
        private ?string $rememberToken = null,
    ) {}

    public function getAuthIdentifier(): int|string
    {
        return $this->id;
    }

    public function getAuthIdentifierName(): string
    {
        return 'id';
    }

    public function getAuthPassword(): string
    {
        return $this->password;
    }

    public function getRememberToken(): ?string
    {
        return $this->rememberToken;
    }

    public function setRememberToken(
        ?string $token,
    ): void {
        $this->rememberToken = $token;
    }

    public function getRememberTokenName(): string
    {
        return 'remember_token';
    }
}
```

## Guards

Guards define how users are authenticated. The package includes two built-in guards.

### SessionGuard

The default guard for web applications. Stores user ID in the session and supports "remember me" functionality:

```php
// Use specific guard
$guard = $this->authManager->guard('session');

// Login with remember me
$guard->login($user, remember: true);

// Check via session
if ($guard->check()) {
    $user = $guard->user();
}
```

Resolve guards through `AuthManager` (or inject `GuardInterface`, which is bound to `AuthManager::guard()`). Guards built this way receive the event dispatcher, the cookie jar, and the remember token manager, so remember-me and [events](#events) work out of the box. If you construct a `SessionGuard` by hand without a cookie jar and token manager, `login($user, remember: true)` throws an `AuthException` instead of silently ignoring the flag.

`SessionGuard` also implements `Marko\Core\Contracts\ResettableInterface`. In a long-running worker (e.g. Swoole, RoadRunner), call `reset()` between requests to clear the guard's cached user so one request's authenticated user is never served to the next (`marko/roadrunner` does this automatically for every resolved `ResettableInterface` service):

```php
use Marko\Core\Contracts\ResettableInterface;

if ($guard instanceof ResettableInterface) {
    $guard->reset();
}
```

`reset()` only clears the cached user --- it does not call `logout()` or otherwise touch the session. The next call to `user()` re-reads the authenticated user from the session as normal.

### TokenGuard

For API authentication via Bearer tokens in the `Authorization` header:

```php
$guard = $this->authManager->guard('token');

// TokenGuard extracts token from Authorization header
// Authorization: Bearer your-api-token
if ($guard->check()) {
    $user = $guard->user();
}
```

### Custom Guards

Implement `GuardInterface` to create custom guards:

```php title="JwtGuard.php"
use Marko\Authentication\AuthenticatableInterface;
use Marko\Authentication\Contracts\GuardInterface;
use Marko\Authentication\Contracts\UserProviderInterface;

class JwtGuard implements GuardInterface
{
    public UserProviderInterface $provider {
        set {
            $this->provider = $value;
        }
    }

    public function check(): bool
    {
        return $this->user() !== null;
    }

    public function guest(): bool
    {
        return !$this->check();
    }

    public function user(): ?AuthenticatableInterface
    {
        // Decode JWT and retrieve user
    }

    public function id(): int|string|null
    {
        return $this->user()?->getAuthIdentifier();
    }

    public function attempt(
        array $credentials,
    ): bool {
        // JWT guards typically don't use attempt()
        return false;
    }

    public function login(
        AuthenticatableInterface $user,
    ): void {
        // Generate and return JWT
    }

    public function loginById(
        int|string $id,
    ): ?AuthenticatableInterface {
        return null;
    }

    public function logout(): void {
        // Invalidate token
    }

    public function getName(): string
    {
        return 'jwt';
    }
}
```

## Remember Me

Pass `remember: true` when logging a user in to keep them signed in after their session expires:

```php title="LoginController.php"
use Marko\Authentication\Contracts\GuardInterface;
use Marko\Authentication\Contracts\UserProviderInterface;
use Marko\Routing\Http\Request;
use Marko\Routing\Http\Response;

class LoginController
{
    public function __construct(
        private GuardInterface $guard,
        private UserProviderInterface $userProvider,
    ) {}

    public function login(
        Request $request,
    ): Response {
        $credentials = [
            'email' => $request->post('email'),
            'password' => $request->post('password'),
        ];
        $user = $this->userProvider->retrieveByCredentials($credentials);

        if ($user === null || !$this->userProvider->validateCredentials($user, $credentials)) {
            return new Response('Invalid credentials', 401);
        }

        $this->guard->login($user, remember: (bool) $request->post('remember'));

        return Response::redirect('/dashboard');
    }
}
```

### How It Works

1. `login($user, remember: true)` generates a random token, stores its SHA-256 hash through `UserProviderInterface::updateRememberToken()`, and queues a `remember_{guard}` cookie (e.g. `remember_session`) holding `{user id}|{plain token}`.
2. `QueuedCookiesMiddleware` attaches the queued cookie to the response with `Response::withCookie()`. The package registers this middleware as global middleware and orders it after the session driver modules, so there is nothing to wire up. Cookies never go through `setcookie()`, so behavior is identical under PHP-FPM and RoadRunner.
3. On a later request with no authenticated session, `user()` reads the cookie, calls `retrieveByRememberToken($id, $hash)` with the **hash** of the cookie's token, verifies it against `getRememberToken()` with a constant-time comparison, and rotates the token (a new cookie is sent) to prevent replay.
4. `logout()` clears the stored token (`updateRememberToken($user, null)`) and sends an expired remember cookie.

### User Provider Requirements

Remember-me stores tokens through your user provider, so the provider and user must persist them:

- `updateRememberToken()` must call `$user->setRememberToken($token)` and save it (typically a nullable `remember_token` column). If the user's `getRememberToken()` does not return the new hash afterwards, `login(..., remember: true)` throws an `AuthException` rather than issuing a cookie that can never be honored.
- `retrieveByRememberToken()` receives the stored hash. Compare it to the stored value with `hash_equals()`.

```php title="UserProvider.php"
public function retrieveByRememberToken(
    int|string $identifier,
    string $token,
): ?AuthenticatableInterface {
    $user = $this->userRepository->find((int) $identifier);
    $storedToken = $user?->getRememberToken();

    if ($storedToken === null || !hash_equals($storedToken, $token)) {
        return null;
    }

    return $user;
}

public function updateRememberToken(
    AuthenticatableInterface $user,
    ?string $token,
): void {
    $user->setRememberToken($token);
    $this->userRepository->save($user);
}
```

### Cookie Configuration

All remember-me options live under `remember` in `config/authentication.php`:

| Key | Default | Description |
|-----|---------|-------------|
| `lifetime` | `43200` | Cookie and token lifetime in minutes (30 days) |
| `cookie.prefix` | `'remember_'` | Cookie name prefix; the guard name is appended (`remember_session`) |
| `cookie.path` | `'/'` | Cookie `Path` attribute |
| `cookie.domain` | `''` | Cookie `Domain` attribute; an empty string omits it |
| `cookie.secure` | `null` | Cookie `Secure` flag; `null` follows `session.cookie.secure` |
| `cookie.http_only` | `true` | Cookie `HttpOnly` flag |
| `cookie.same_site` | `'Lax'` | Cookie `SameSite` attribute (`Lax`, `Strict` or `None`; `None` requires `secure`) |

### Cookie Jar

The guard reads and writes cookies through `CookieJarInterface`, which is bound as a singleton to `Marko\Authentication\Cookie\RequestCookieJar`. The jar reads from the current request and queues writes until `QueuedCookiesMiddleware` attaches them to the response. Writing a cookie when no HTTP request is being handled (for example, `login(..., remember: true)` from a CLI command) throws an `AuthException`. The jar implements `ResettableInterface`, so long-running workers clear its request and queue between requests.

## Middleware

### AuthMiddleware

Protects routes by requiring authentication:

```php title="DashboardController.php"
use Marko\Authentication\Middleware\AuthMiddleware;
use Marko\Routing\Attributes\Get;
use Marko\Routing\Attributes\Middleware;

class DashboardController
{
    #[Get('/dashboard')]
    #[Middleware(AuthMiddleware::class)]
    public function index(): Response
    {
        // Only authenticated users reach here
    }
}
```

When the request is not authenticated, `AuthMiddleware` does one of two things:

- **Redirects** to `redirectTo` (default `/login`) when the guard is stateful, such as `SessionGuard`. A redirect is a real response, not an error.
- **Throws `HttpException::unauthorized()`** when `redirectTo` is `null`, and always for `TokenGuard`, because API clients can't follow a login redirect.

The routing pipeline renders the thrown `401` through [`ExceptionRenderer`](/docs/packages/routing/#errors-and-http-exceptions), so the format comes from the request, not from the guard. It is JSON when the `Accept` header asks for `application/json` or a `+json` type (or when the request has a JSON `Content-Type` and no `Accept`), and a minimal HTML page otherwise:

```json
{"message": "Unauthorized."}
```

To disable the redirect for a web guard, register a binding with `redirectTo: null`:

```php title="module.php"
use Marko\Authentication\AuthManager;
use Marko\Authentication\Middleware\AuthMiddleware;
use Marko\Core\Container\ContainerInterface;

// In 'bindings'
AuthMiddleware::class => fn (ContainerInterface $container): AuthMiddleware => new AuthMiddleware(
    auth: $container->get(AuthManager::class),
    redirectTo: null,
),
```

To change how the `401` looks (a branded page, a different JSON shape), replace `ExceptionRenderer` with a `#[Preference]`. See [Custom error pages](/docs/packages/routing/#custom-error-pages). Authorization failures from `#[Can]` and `Gate::authorize()` use the same renderer; see [Failure Responses](/docs/packages/authorization/#failure-responses).

### GuestMiddleware

Restricts routes to unauthenticated users (e.g., login pages):

```php title="LoginController.php"
use Marko\Authentication\Middleware\GuestMiddleware;

class LoginController
{
    #[Get('/login')]
    #[Middleware(GuestMiddleware::class)]
    public function show(): Response
    {
        // Only guests can view the login page
    }
}
```

Authenticated users are redirected to a configured path (default: `/`).

## Events

The auth package dispatches [events](/docs/packages/events/) during the authentication lifecycle. Create observers to react to these events.

`SessionGuard` dispatches `LoginEvent`, `LogoutEvent` and `FailedLoginEvent` through core's `EventDispatcherInterface`, which `AuthManager` passes to every session guard it builds. Any guard you get from `AuthManager::guard()` or by injecting `GuardInterface` fires them. `TokenGuard` is stateless (no login or logout) and dispatches no events.

### LoginEvent

Dispatched when a user successfully logs in:

```php title="LogLoginObserver.php"
use Marko\Authentication\Event\LoginEvent;
use Marko\Core\Attributes\Observer;

#[Observer(LoginEvent::class)]
class LogLoginObserver
{
    public function handle(
        LoginEvent $event,
    ): void {
        $user = $event->getUser();
        $guard = $event->getGuard();
        $remember = $event->getRemember();

        // Log the login, update last_login timestamp, etc.
    }
}
```

### LogoutEvent

Dispatched when a user logs out:

```php title="LogLogoutObserver.php"
use Marko\Authentication\Event\LogoutEvent;
use Marko\Core\Attributes\Observer;

#[Observer(LogoutEvent::class)]
class LogLogoutObserver
{
    public function handle(
        LogoutEvent $event,
    ): void {
        $user = $event->getUser();
        $guard = $event->getGuard();

        // Clean up user session data, log activity, etc.
    }
}
```

### FailedLoginEvent

Dispatched when authentication fails (credentials not found or invalid):

```php title="LogFailedLoginObserver.php"
use Marko\Authentication\Event\FailedLoginEvent;
use Marko\Core\Attributes\Observer;

#[Observer(FailedLoginEvent::class)]
class LogFailedLoginObserver
{
    public function handle(
        FailedLoginEvent $event,
    ): void {
        $credentials = $event->getCredentials(); // Password removed for security
        $guard = $event->getGuard();

        // Log failed attempt, implement rate limiting, alert on suspicious activity
    }
}
```

### PasswordResetEvent

Dispatched when a user's password is reset:

```php title="NotifyPasswordResetObserver.php"
use Marko\Authentication\Event\PasswordResetEvent;
use Marko\Core\Attributes\Observer;

#[Observer(PasswordResetEvent::class)]
class NotifyPasswordResetObserver
{
    public function handle(
        PasswordResetEvent $event,
    ): void {
        $user = $event->getUser();

        // Send confirmation email, invalidate other sessions, etc.
    }
}
```

## API Reference

### AuthManager

```php
public function guard(?string $name = null): GuardInterface;
public function useGuard(string $name, GuardInterface $guard): void;
public function check(): bool;
public function user(): ?AuthenticatableInterface;
public function id(): int|string|null;
public function attempt(array $credentials): bool;
public function logout(): void;
```

`useGuard()` puts a guard instance in place for a guard name, replacing any guard already built for it; `guard($name)` returns it from then on. The [marko/testing](/docs/packages/testing/) HTTP test client's `actingAs()` uses it to authenticate a user without a login request.

### GuardInterface

```php
public function check(): bool;
public function guest(): bool;
public function user(): ?AuthenticatableInterface;
public function id(): int|string|null;
public function attempt(array $credentials): bool;
public function login(AuthenticatableInterface $user): void;
public function loginById(int|string $id): ?AuthenticatableInterface;
public function logout(): void;
public UserProviderInterface $provider { set; }
public function getName(): string;
```

### AuthenticatableInterface

```php
public function getAuthIdentifier(): int|string;
public function getAuthIdentifierName(): string;
public function getAuthPassword(): string;
public function getRememberToken(): ?string;
public function setRememberToken(?string $token): void;
public function getRememberTokenName(): string;
```

### UserProviderInterface

```php
public function retrieveById(int|string $identifier): ?AuthenticatableInterface;
public function retrieveByCredentials(array $credentials): ?AuthenticatableInterface;
public function validateCredentials(AuthenticatableInterface $user, array $credentials): bool;
public function retrieveByRememberToken(int|string $identifier, string $token): ?AuthenticatableInterface;
public function updateRememberToken(AuthenticatableInterface $user, ?string $token): void;
```

`retrieveByRememberToken()` receives the SHA-256 hash of the cookie's token, the same value previously passed to `updateRememberToken()`.

### SessionGuard

```php
public function login(AuthenticatableInterface $user, bool $remember = false): void;
```

### CookieJarInterface

```php
public function get(string $name): ?string;
public function set(string $name, string $value, int $minutes = 0): void;
public function delete(string $name): void;
```

### RequestCookieJar

```php
public function setRequest(Request $request): void;
public function pullQueuedCookies(): array; // array<int, Cookie>
public function reset(): void;
```
