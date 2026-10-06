---
title: marko/authentication
description: Session and token-based authentication — guards protect routes, events track activity, middleware controls access.
---

Session and token-based authentication — guards protect routes, events track activity, middleware controls access. The auth package ships `SessionGuard` for web applications and a guard driver registry for everything else: install [marko/authentication-token](/docs/packages/authentication-token/) for API tokens, or register your own driver. Configure multiple guards, implement custom user providers, and react to authentication events via observers.

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
        'users' => [], // no 'class': the app's UserProviderInterface binding
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

Each guard's `driver` picks how it is built. `session` is built in. The shipped `token` guard needs [marko/authentication-token](/docs/packages/authentication-token/); until it is installed, resolving that guard throws an `AuthException` that tells you to install it. See [Guard Drivers](#guard-drivers).

Each guard's `provider` names an entry in `providers`. See [User Providers](#user-providers).

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

Guards define how users are authenticated. `AuthManager::guard($name)` builds the guard configured under `authentication.guards.{name}` with its `driver`, and caches it for the rest of the request. Called without a name, it builds `authentication.default.guard`.

A guard name that isn't a key of `authentication.guards` throws an `AuthException` naming the guard and listing the configured ones. A misspelt `authentication.default.guard` (or a name passed to `guard()`) never falls back to a session guard. A guard entry without a `driver` throws too:

```php
$this->authManager->guard('admni');
// AuthException: Guard 'admni' is not defined in authentication.guards
// Context: AuthManager was asked for guard 'admni'. Configured guards: session, token
```

`GuardInterface` is bound to `AuthManager::guard()` with the default name, so a bad `authentication.default.guard` fails every consumer of `GuardInterface` the same way. When routes use `#[Can]`, the [authorization boot check](/docs/packages/authorization/#cost-on-routes-without-can) catches it before the first request.

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

### Token Guards

API token authentication lives in [marko/authentication-token](/docs/packages/authentication-token/). Installing it registers the `token` driver, so the `token` guard in the default config (or any guard with `'driver' => 'token'`) resolves to `Marko\AuthenticationToken\Guard\TokenGuard`:

```php
$guard = $this->authManager->guard('token');

// Authorization: Bearer your-api-token
if ($guard->check()) {
    $user = $guard->user();
}
```

### Stateless Guards

A guard that authenticates each request from credentials the request carries implements `Marko\Authentication\Contracts\StatelessGuardInterface`. It adds one method to `GuardInterface`, `getChallenge()`, which returns the `WWW-Authenticate` challenge (for example `Bearer`). Neither `AuthMiddleware` nor [`AdminAuthMiddleware`](/docs/packages/admin-auth/#protecting-admin-routes) redirects a guest on a stateless guard. Every `401` the framework sends for a guest (from `AuthMiddleware`, from `#[Can]` in [marko/authorization](/docs/packages/authorization/#failure-responses), and from `AdminAuthMiddleware`) carries the guard's challenge, because each one is built by `UnauthenticatedException::forGuard()`. Throw the same exception from your own middleware so its `401` looks identical:

```php title="ApiKeyMiddleware.php"
use Marko\Authentication\Exceptions\UnauthenticatedException;

if (!$guard->check()) {
    throw UnauthenticatedException::forGuard($guard); // 401, plus WWW-Authenticate for a stateless guard
}
```

A stateless guard's `attempt()`, `login()`, `loginById()` and `logout()` must throw and explain what to do instead, never silently do nothing.

### Guard Drivers

`AuthManager` builds guards through `Marko\Authentication\Guard\GuardDriverRegistry`, a singleton that maps driver names to factories. Every guard entry must set `driver`; a missing, empty or non-string `driver` throws an `AuthException` naming the guard (there is no implicit `session` default). It checks the registry first, then the built-in `session` driver. Any other driver name throws an `AuthException` that lists the drivers available.

Register a driver from your module's `boot` callback. The factory receives the guard name, that guard's config array and the user provider, and must return a guard whose `getName()` is the guard name (`AuthManager` throws otherwise). Resolve heavy dependencies inside the factory so booting stays cheap:

```php title="module.php"
use Marko\Authentication\Contracts\GuardInterface;
use Marko\Authentication\Contracts\UserProviderInterface;
use Marko\Authentication\Guard\GuardDriverRegistry;
use Marko\Core\Container\ContainerInterface;

return [
    'boot' => function (GuardDriverRegistry $guardDriverRegistry, ContainerInterface $container): void {
        $guardDriverRegistry->extend(
            'jwt',
            fn (string $name, array $config, UserProviderInterface $provider): GuardInterface => new JwtGuard(
                decoder: $container->get(JwtDecoder::class),
                provider: $provider,
                name: $name,
                header: $config['header'] ?? 'Authorization',
            ),
        );
    },
];
```

```php title="config/authentication.php"
'guards' => [
    'partner-api' => ['driver' => 'jwt', 'provider' => 'users', 'header' => 'X-Partner-Token'],
],
```

Registering a driver name again replaces the earlier factory, so a later module (your app, for example) can override `token` or even `session`.

### Custom Guards

A guard whose credentials can be scoped to a subset of what the user may do (an API token issued with abilities, for example) implements `Marko\Authentication\Contracts\AbilityScopedGuardInterface`, which adds `hasAbility(string $ability): bool`. The [authorization Gate](/docs/packages/authorization/#api-token-abilities) denies an authenticated user any ability the guard's `hasAbility()` rejects, so `#[Can]` respects token scopes.

Implement `GuardInterface` to create custom guards, then register them as a driver. A stateless guard implements `StatelessGuardInterface` instead and throws from the stateful methods:

```php title="JwtGuard.php"
use Marko\Authentication\AuthenticatableInterface;
use Marko\Authentication\Contracts\StatelessGuardInterface;
use Marko\Authentication\Contracts\UserProviderInterface;
use Marko\Authentication\Exceptions\AuthException;

class JwtGuard implements StatelessGuardInterface
{
    public function __construct(
        private JwtDecoder $decoder,
        public UserProviderInterface $provider {
            set {
                $this->provider = $value;
            }
        },
        private string $name = 'jwt',
        private string $header = 'Authorization',
    ) {}

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
        throw new AuthException(
            message: "Cannot call attempt() on JWT guard '$this->name': it is stateless",
            suggestion: 'Issue a JWT from your login endpoint instead',
        );
    }

    // login(), loginById() and logout() throw the same way

    public function getName(): string
    {
        return $this->name;
    }

    public function getChallenge(): string
    {
        return 'Bearer';
    }
}
```

## User Providers

A user provider (`UserProviderInterface`) loads users for a guard. Each guard gets its own provider, so one application can authenticate different kinds of users, for example customers on the frontend and admins in the admin panel, without one store standing in for the other.

`AuthManager` picks a guard's provider with `UserProviderResolver`:

1. The guard's `provider` key names an entry in `authentication.providers`. A guard without one uses `authentication.default.provider`.
2. The entry's `class` key names a `UserProviderInterface` implementation, resolved from the container.
3. A guard with no provider name, a provider entry without a `class`, or an application with no `authentication.providers` at all uses the container's `UserProviderInterface` binding.

A single-provider application therefore needs nothing more than the binding:

```php title="app/web/module.php"
use App\Web\Auth\CustomerProvider;
use Marko\Authentication\Contracts\UserProviderInterface;

return [
    'bindings' => [
        UserProviderInterface::class => CustomerProvider::class,
    ],
];
```

Give a guard its own provider with a `class`:

```php title="config/authentication.php"
return [
    'guards' => [
        'session' => ['driver' => 'session', 'provider' => 'users'],
        'partner' => ['driver' => 'session', 'provider' => 'partners'],
    ],
    'providers' => [
        'users' => [],
        'partners' => ['class' => App\Partner\Auth\PartnerProvider::class],
    ],
];
```

Each provider is built once and shared by every guard that names it. A guard naming a provider missing from `authentication.providers`, or a `class` that does not implement `UserProviderInterface`, throws an `AuthException`:

```php
// AuthException: User provider 'partners' is not defined in authentication.providers
// Context: Guard 'partner' uses provider 'partners'. Configured providers: users
```

Every session guard keeps its user in its own session key, `auth_{guard}_user_id`, and its own remember-me cookie, `remember_{guard}`. Logging in on one guard never authenticates another. [marko/admin-auth](/docs/packages/admin-auth/#the-admin-guard) uses this to ship a separate `admin` guard backed by `AdminUserProvider`.

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

### Time and Testing

Remember-token expiry (`RememberTokenManager`) and remember-cookie expiry (`RequestCookieJar`) read the current time through the PSR-20 `Psr\Clock\ClockInterface` from [`marko/clock`](/docs/packages/clock/), never `time()`. Both take the clock as a constructor parameter and the container injects it. In tests, pass a [`FakeClock`](/docs/packages/testing/#fakeclock) to check the lifetime boundary exactly, without sleeping:

```php
use Marko\Authentication\Token\RememberTokenManager;
use Marko\Testing\Fake\FakeClock;

$clock = new FakeClock('2026-01-01 12:00:00 UTC');
$manager = new RememberTokenManager($clock, lifetimeMinutes: 60);
$createdAt = $clock->now();

$clock->travel('+60 minutes');
$manager->isExpired($createdAt); // false

$clock->travel('+1 second');
$manager->isExpired($createdAt); // true
```

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

- **Redirects** to `redirectTo` (default `/login`) when the guard is stateful, such as `SessionGuard`, and the request does not want JSON. A redirect is a real response, not an error.
- **Throws a `401` `UnauthenticatedException`** (an `HttpException`) otherwise: when `redirectTo` is `null`, when the request wants JSON (`Request::wantsJson()`, whatever the guard), and always for a [stateless guard](#stateless-guards) such as the token guard, because API clients can't follow a login redirect. A stateless guard's `401` also carries its `WWW-Authenticate` challenge (`WWW-Authenticate: Bearer` for the token guard).

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

`SessionGuard` dispatches `LoginEvent`, `LogoutEvent` and `FailedLoginEvent` through core's `EventDispatcherInterface`, which `AuthManager` passes to every session guard it builds. Any guard you get from `AuthManager::guard()` or by injecting `GuardInterface` fires them. Stateless guards have no login or logout, so they never fire these events. The token guard has its own token lifecycle events instead; see [marko/authentication-token events](/docs/packages/authentication-token/#events).

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
public function reset(): void;
```

`AuthManager` implements `ResettableInterface`: `reset()` clears the per-request state (the resolved user) of every guard it has built or been given, so a long-running worker never serves one request with the previous request's user. `useGuard()` puts a guard instance in place for a guard name, replacing any guard already built for it; `guard($name)` returns it from then on, even for a name that isn't in `authentication.guards`. The [marko/testing](/docs/packages/testing/) HTTP test client's `actingAs()` uses it to authenticate a user without a login request.

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

### StatelessGuardInterface

Extends `GuardInterface`:

```php
public function getChallenge(): string;
```

### UnauthenticatedException

Extends `Marko\Routing\Exceptions\HttpException`. Its message is `Unauthorized.`, and the guard name goes in the log-only context:

```php
public static function forGuard(GuardInterface $guard): self; // 401; adds WWW-Authenticate: getChallenge() for a StatelessGuardInterface
```

### GuardDriverRegistry

```php
public function extend(string $driver, Closure $factory): void; // Closure(string $name, array $config, UserProviderInterface $provider): GuardInterface
public function has(string $driver): bool;
public function drivers(): array; // list<string>
public function create(string $driver, string $name, array $config, UserProviderInterface $provider): ?GuardInterface;
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

### UserProviderResolver

```php
public function forGuard(string $guard, array $guardConfig): UserProviderInterface;
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

`retrieveByCredentials()` and `validateCredentials()` receive request input as-is, so a provider must treat non-string `email` or `password` values (such as `email[]=x`) as a failed login rather than passing them to typed methods. When `retrieveByCredentials()` finds no usable account (unknown or inactive), it should call `PasswordHasherInterface::verifyDummy()` before returning `null`, so a failed login costs one password check either way and response timing does not reveal which accounts exist. `AdminUserProvider` in [admin-auth](/docs/packages/admin-auth/) does both.

### PasswordHasherInterface

```php
public function hash(string $password): string;
public function verify(string $password, string $hash): bool;
public function needsRehash(string $hash): bool;
public function verifyDummy(string $password): void;
```

`verifyDummy()` checks the password against a fixed dummy hash of the configured cost and discards the result. `BcryptPasswordHasher` builds that dummy hash at its own cost, so the check takes as long as verifying a real stored password.

Bcrypt only uses the first 72 bytes of a password and cannot hash a NUL (`\0`) byte. Rather than silently truncate, `BcryptPasswordHasher::hash()` throws `InvalidPasswordException` for a password longer than 72 bytes (`BcryptPasswordHasher::MAX_PASSWORD_BYTES`) or containing a NUL byte. `verify()` returns `false` for such a password and never throws; it still runs the dummy check first, so rejecting an oversize password at login takes as long as checking a real one. The limit is in bytes, not characters, so validate passwords with `strlen()` on registration and password change to show a form error instead of an exception:

```php
use Marko\Authentication\Hashing\BcryptPasswordHasher;

if (strlen($password) > BcryptPasswordHasher::MAX_PASSWORD_BYTES || str_contains($password, "\0")) {
    // Reject with a validation error
}
```

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

### RememberTokenManager

```php
public function __construct(ClockInterface $clock, ?int $lifetimeMinutes = null);
public function generate(): string;
public function hash(string $token): string;
public function validate(string $token, string $storedHash): bool;
public function isExpired(DateTimeImmutable $createdAt): bool;
```

### RequestCookieJar

```php
public function __construct(AuthConfig $config, ClockInterface $clock);
public function setRequest(Request $request): void;
public function pullQueuedCookies(): array; // array<int, Cookie>
public function reset(): void;
```
