---
title: marko/authorization
description: Gates, policies, and the #[Can] attribute -- control who can do what with expressive, testable authorization checks.
---

Gates, policies, and the `#[Can]` attribute --- control who can do what with expressive, testable authorization checks. Define abilities with closures via the Gate, or organize them into policy classes mapped to entities. Use `#[Can]` on controllers or their methods to enforce permissions through middleware that runs on every route once the package is installed. Denials throw `AuthorizationException` with clear context.

## Installation

```bash
composer require marko/authorization
```

## Usage

### Defining Abilities

Register abilities as closures on the Gate:

```php
use Marko\Authorization\AuthorizableInterface;
use Marko\Authorization\Contracts\GateInterface;

readonly class AuthorizationBootstrap
{
    public function __construct(
        private GateInterface $gate,
    ) {}

    public function boot(): void
    {
        $this->gate->define(
            'edit-settings',
            fn (?AuthorizableInterface $user) => $user?->can('admin', true) ?? false,
        );
    }
}
```

### Checking Abilities

```php
use Marko\Authorization\Contracts\GateInterface;

readonly class SettingsController
{
    public function __construct(
        private GateInterface $gate,
    ) {}

    public function update(): void
    {
        if ($this->gate->denies('edit-settings')) {
            // handle unauthorized
        }

        // proceed with update
    }
}
```

Use `authorize()` to throw on denial:

```php
$this->gate->authorize('edit-settings');
// Throws AuthorizationException if denied
```

You don't need to catch it in a controller. `AuthorizationException` implements `HttpExceptionInterface`, so the routing pipeline turns an uncaught denial into a `403` (see [Failure Responses](#failure-responses)).

### Using Policies

Policies group authorization logic per entity. Create a policy class with methods named after abilities:

```php
use Marko\Authorization\AuthorizableInterface;

class PostPolicy
{
    public function update(
        ?AuthorizableInterface $user,
        Post $post,
    ): bool {
        return $user !== null && $user->getAuthIdentifier() === $post->authorId;
    }

    public function delete(
        ?AuthorizableInterface $user,
        Post $post,
    ): bool {
        return $user !== null && $user->getAuthIdentifier() === $post->authorId;
    }
}
```

Register the policy:

```php
$this->gate->policy(
    Post::class,
    PostPolicy::class,
);
```

Check against the policy by passing the entity:

```php
$this->gate->authorize('update', $post);
```

### The #[Can] Attribute

Add `#[Can]` to a controller method to require an ability before the action runs:

```php
use Marko\Authorization\Attributes\Can;
use Marko\Routing\Attributes\Get;
use Marko\Routing\Attributes\Post as PostRoute;
use Marko\Routing\Http\Response;

class PostController
{
    #[Get('/admin/stats')]
    #[Can('view-stats')]
    public function stats(): Response
    {
        // Only reachable if the gate allows 'view-stats'
    }

    #[PostRoute('/posts')]
    #[Can(ability: 'create', entityClass: Post::class)]
    public function store(): Response
    {
        // Only reachable if PostPolicy::create() allows it
    }
}
```

Installing `marko/authorization` registers `AuthorizationMiddleware` as global middleware, so you don't attach it to routes yourself. For every matched route it reads `#[Can]` from the controller action and checks the Gate:

- **No `#[Can]`**: the request passes through untouched.
- **Not logged in**: throws `HttpException::unauthorized()`, which renders as `401 Unauthorized`.
- **Logged in but denied**: throws `AuthorizationException`, which renders as `403 Forbidden`.

The middleware never builds these responses itself; see [Failure Responses](#failure-responses).

The middleware checks authentication with the same guard the Gate uses, so the guard named by `authorization.default_guard` decides whether the request gets a `401`. If that value is `null`, the authentication default guard is used.

Don't also list `AuthorizationMiddleware` in a route's `middleware` array. It already runs globally, so it would only check the same ability a second time.

#### Class-Level `#[Can]`

Put `#[Can]` on the controller class to protect every action in it. A `#[Can]` on a method replaces the class-level one for that action:

```php
use Marko\Authorization\Attributes\Can;
use Marko\Routing\Attributes\Get;
use Marko\Routing\Http\Response;

#[Can('admin.access')]
class AdminController
{
    #[Get('/admin')]
    public function dashboard(): Response
    {
        // Requires 'admin.access'
    }

    #[Get('/admin/reports')]
    #[Can('admin.reports')]
    public function reports(): Response
    {
        // Requires 'admin.reports' only
    }
}
```

PHP attributes are not inherited, so a class-level `#[Can]` applies only to the class that declares it, not to its subclasses. Repeat it on a subclass that needs the same protection.

#### Ordering with Sessions

The guard reads the logged-in user from the session, so authorization runs after the session middleware. `marko/authorization` declares `sequence.after` on `marko/session-file` and `marko/session-database`, the session drivers that register `SessionMiddleware` globally. Drivers that aren't installed are ignored.

#### Class-String vs Instance Checks

With `entityClass`, `#[Can]` passes the class name (for example `Post::class`) to the Gate, not an entity. The Gate resolves the policy from the class name, so this fits abilities that don't need a specific record, such as `create` or `viewAny`:

```php
use Marko\Authorization\AuthorizableInterface;

class PostPolicy
{
    public function create(
        ?AuthorizableInterface $user,
        string $postClass,
    ): bool {
        return $user !== null;
    }
}
```

Abilities that depend on a specific record, such as `update` or `delete` on one post, need the loaded entity. Marko has no route model binding (explicit over implicit), so load the entity in the controller and call `authorize()` yourself:

```php
use Marko\Authorization\Contracts\GateInterface;
use Marko\Routing\Attributes\Put;
use Marko\Routing\Http\Response;

readonly class PostController
{
    public function __construct(
        private GateInterface $gate,
        private PostRepository $posts,
    ) {}

    #[Put('/posts/{id}')]
    public function update(
        int $id,
    ): Response {
        $post = $this->posts->find($id);
        $this->gate->authorize('update', $post);

        // proceed with update
    }
}
```

### Failure Responses

Authorization failures are thrown as exceptions that implement `Marko\Core\Exceptions\HttpExceptionInterface`. The routing pipeline renders them through [`ExceptionRenderer`](/docs/packages/routing/#errors-and-http-exceptions), in the same way as every other HTTP error:

| Failure | Thrown by | Exception | Status |
|---|---|---|---|
| `#[Can]` route, no authenticated user | `AuthorizationMiddleware` | `Marko\Routing\Exceptions\HttpException` | `401` |
| `#[Can]` route, ability denied | `AuthorizationMiddleware` | `Marko\Authorization\Exceptions\AuthorizationException` | `403` |
| `$gate->authorize()` denied in a controller or service | `Gate` | `Marko\Authorization\Exceptions\AuthorizationException` | `403` |

The renderer picks the format from the request. It sends JSON when the `Accept` header asks for `application/json` or any `+json` type (such as `application/vnd.api+json`), or when the request has a JSON `Content-Type` and no `Accept` header. Otherwise it sends a minimal HTML page:

```json
{"message": "Unauthorized."}
{"message": "Forbidden."}
```

The `403` body never contains the ability or the resource name, because they can leak internals. Both stay on the exception for logging, through `getAbility()` and `getResource()`.

A `Gate` has no notion of a guest. A denied `authorize()` call is always a `403`, even when nobody is logged in. Use `#[Can]` (or [`AuthMiddleware`](/docs/packages/authentication/#authmiddleware)) when a guest should get a `401`.

To render branded error pages or a different JSON shape, replace `ExceptionRenderer` with a `#[Preference]`. See [Custom error pages](/docs/packages/routing/#custom-error-pages) in the routing docs. Authorization failures go through it like any other HTTP error, so nothing authorization-specific is needed.

Mistakes in policy setup are a different case: registering two policies for one entity, or checking an ability the policy has no method for. These throw `PolicyException`, which does **not** implement `HttpExceptionInterface`. It reaches your error handler as a `500` with the full message, instead of being hidden behind a `403`.

### Implementing AuthorizableInterface

Your user entity must implement `AuthorizableInterface`, which extends [`AuthenticatableInterface`](/docs/packages/authentication/):

```php
use Marko\Authorization\AuthorizableInterface;

class User implements AuthorizableInterface
{
    public function can(
        string $ability,
        mixed ...$arguments,
    ): bool {
        // Check ability via gate or custom logic
    }

    public function getAuthIdentifier(): int|string
    {
        return $this->id;
    }

    public function getAuthPassword(): string
    {
        return $this->passwordHash;
    }
}
```

## API Reference

### GateInterface

```php
interface GateInterface
{
    public function define(string $ability, callable $callback): void;
    public function allows(string $ability, mixed ...$arguments): bool;
    public function denies(string $ability, mixed ...$arguments): bool;
    public function authorize(string $ability, mixed ...$arguments): bool;
    public function policy(string $entityClass, string $policyClass): void;
}
```

### AuthorizableInterface

```php
interface AuthorizableInterface extends AuthenticatableInterface
{
    public function can(string $ability, mixed ...$arguments): bool;
}
```

### #[Can] Attribute

```php
#[Can(ability: 'edit', entityClass: Post::class)]  // With entity
#[Can(ability: 'access-dashboard')]                  // Without entity
```

Targets classes and methods. A method-level `#[Can]` overrides a class-level one.

### AuthorizationMiddleware

Registered as global middleware by `module.php`. It reads the matched controller and action from the request.

```php
class AuthorizationMiddleware implements MiddlewareInterface
{
    public function __construct(GateInterface $gate, GuardInterface $guard);
    public function handle(Request $request, callable $next): Response;
}
```

### AuthorizationException

Rendered as `403` with `{"message": "Forbidden."}`. The ability and resource are never sent to the client.

```php
class AuthorizationException extends MarkoException implements HttpExceptionInterface
{
    public function __construct(
        string $message = 'Forbidden',
        string $ability = '',
        string $resource = '',
        string $context = '',
        string $suggestion = '',
        ?Throwable $previous = null,
    );

    public function getAbility(): string;
    public function getResource(): string;
    public function getStatusCode(): int;          // 403
    public function getHeaders(): array;           // []
    public function getResponseData(): array;      // ['message' => 'Forbidden.']

    public static function forbidden(string $ability, string $resource): self;
}
```

### PolicyException

Thrown for policy misconfiguration. It is not an HTTP exception, so it surfaces as a `500`.

```php
class PolicyException extends MarkoException
{
    public static function duplicatePolicy(string $entityClass, string $policyClass, string $existing): self;
    public static function missingMethod(string $policyClass, string $ability): self;
}
```
