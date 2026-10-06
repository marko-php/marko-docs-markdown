---
title: marko/admin-auth
description: Admin authentication and role-based authorization --- manages admin users, roles, permissions, and access control for the admin panel.
---

Admin authentication and role-based authorization --- manages admin users, roles, permissions, and access control for the admin panel. The package provides an `AdminUserProvider` that integrates with the [authentication](/docs/packages/authentication/) system, a `PermissionRegistry` for declaring and matching permissions (including wildcards), role and permission entities with repository interfaces, and `AdminAuthMiddleware` that enforces `#[RequiresPermission]` checks on controller methods. Super admin roles bypass all permission checks.

## Installation

```bash
composer require marko/admin-auth
```

## Usage

### Protecting Admin Routes

Add `AdminAuthMiddleware` to controller methods or classes to require authentication:

```php title="CatalogController.php"
use Marko\AdminAuth\Middleware\AdminAuthMiddleware;
use Marko\Routing\Attributes\Get;
use Marko\Routing\Attributes\Middleware;

class CatalogController
{
    #[Get('/admin/catalog/products')]
    #[Middleware(AdminAuthMiddleware::class)]
    public function index(): Response
    {
        // Only authenticated admin users reach here
    }
}
```

When no admin is logged in, `AdminAuthMiddleware` does one of two things:

- **Redirects** to `{prefix}/login` (`/admin/login` by default) when the guard is stateful (the session guard) and the request does not want JSON.
- **Throws a `401` `UnauthenticatedException`** (an `HttpException` from `marko/authentication`) in every other case: when the request wants JSON (`Request::wantsJson()`: an `Accept` header with `application/json` or a `+json` type), and always when the guard is stateless (`StatelessGuardInterface`, such as the token guard), whatever the `Accept` header, because an API client can't follow a login redirect. On a stateless guard the `401` carries the guard's `WWW-Authenticate` challenge (`Bearer` for the token guard), the same response [`AuthMiddleware`](/docs/packages/authentication/) sends.

`AdminAuthMiddleware` uses the default guard (`GuardInterface`), so an application whose default guard is the token guard, such as a headless admin served through [`marko/admin-api`](/docs/packages/admin-api/), never redirects.

The routing pipeline renders the thrown `401` through [`ExceptionRenderer`](/docs/packages/routing/#errors-and-http-exceptions), so it looks like every other HTTP error in the application:

```json
{"message": "Unauthorized."}
```

### Requiring Permissions

Use `#[RequiresPermission]` to enforce specific permissions on a route. `AdminAuthMiddleware` reads the attribute from the matched controller method via reflection and throws a `403` `HttpException` when the authenticated user lacks the required permission, or is not an admin user at all. Super admin roles bypass this check.

`ExceptionRenderer` renders the `403` as JSON for requests that ask for it and as the application's HTML error page otherwise:

```json
{"message": "Forbidden."}
```

The required permission is never sent to the client. It is only in the exception's context, for logs. To render admin denials differently (a branded page, another JSON shape), replace `ExceptionRenderer` with a `#[Preference]`; see [Custom error pages](/docs/packages/routing/#custom-error-pages). Admin denials go through it like any other HTTP error.

```php title="ProductController.php"
use Marko\AdminAuth\Attributes\RequiresPermission;
use Marko\AdminAuth\Middleware\AdminAuthMiddleware;
use Marko\Routing\Attributes\Get;
use Marko\Routing\Attributes\Middleware;

class ProductController
{
    #[Get('/admin/catalog/products')]
    #[Middleware(AdminAuthMiddleware::class)]
    #[RequiresPermission(permission: 'catalog.products.view')]
    public function index(): Response
    {
        // Only admin users with 'catalog.products.view' permission
    }
}
```

### Registering Permissions

Modules register their permissions via `PermissionRegistryInterface`:

```php title="CatalogPermissions.php"
use Marko\AdminAuth\Contracts\PermissionRegistryInterface;

readonly class CatalogPermissions
{
    public function __construct(
        private PermissionRegistryInterface $permissionRegistry,
    ) {}

    public function register(): void
    {
        $this->permissionRegistry->register(
            'catalog.products.view',
            'View Products',
            'Catalog',
        );
        $this->permissionRegistry->register(
            'catalog.products.edit',
            'Edit Products',
            'Catalog',
        );
    }
}
```

`marko/admin-auth` binds `PermissionRegistryInterface` to `PermissionRegistry` as a shared singleton, so you don't bind it yourself. Every class that injects the interface gets the same instance: your registration classes, `AdminAuthMiddleware` and `marko/admin-api`'s `SectionController`. A permission registered through one of them is visible to all of them.

To replace the registry, put a `#[Preference]` on a class that implements the interface, and have it replace the interface rather than `PermissionRegistry`. The container checks preferences for the type being injected, which is the interface, so a Preference on the concrete class is never used. The replacement is still shared:

```php title="app/admin/src/AppPermissionRegistry.php"
use Marko\AdminAuth\Contracts\PermissionRegistryInterface;
use Marko\AdminAuth\PermissionRegistry;
use Marko\Core\Attributes\Preference;

#[Preference(replaces: PermissionRegistryInterface::class)]
class AppPermissionRegistry extends PermissionRegistry
{
    // Override register(), all(), getByGroup() or matches() as needed
}
```

### Wildcard Permissions

Permissions support wildcard matching. A role with `catalog.*` can access any `catalog.` permission:

```php
use Marko\AdminAuth\Contracts\PermissionRegistryInterface;

$permissionRegistry->matches('catalog.*', 'catalog.products.view');  // true
$permissionRegistry->matches('catalog.*', 'catalog.products.edit');   // true
$permissionRegistry->matches('*', 'anything.here');                   // true
$permissionRegistry->matches('catalog.products.*', 'catalog.orders'); // false
```

### Checking Permissions in Code

`AdminUserInterface` provides methods for checking permissions and roles:

```php title="OrderService.php"
use Marko\AdminAuth\Entity\AdminUserInterface;
use Marko\Routing\Exceptions\HttpException;

class OrderService
{
    public function cancel(
        AdminUserInterface $adminUser,
        int $orderId,
    ): void {
        if (!$adminUser->hasPermission('orders.cancel')) {
            throw HttpException::forbidden('Cannot cancel orders.');
        }

        if ($adminUser->hasRole('super-admin')) {
            // super admin bypass
        }
    }
}
```

### Admin User Entity

`AdminUser` implements `AdminUserInterface` and integrates with the [authentication](/docs/packages/authentication/) system:

```php title="DashboardController.php"
use Marko\AdminAuth\Entity\AdminUserInterface;
use Marko\Authentication\Contracts\GuardInterface;

readonly class DashboardController
{
    public function __construct(
        private GuardInterface $guard,
    ) {}

    public function index(): Response
    {
        $user = $this->guard->user();

        if ($user instanceof AdminUserInterface) {
            $name = $user->getName();
            $roles = $user->getRoles();
            $permissions = $user->getPermissionKeys();
        }
    }
}
```

### Events

`RoleRepository` and `AdminUserRepository` dispatch `RoleCreated`, `RoleUpdated`, `RoleDeleted`, `AdminUserCreated` and `AdminUserUpdated` after a save or delete. `AdminUserDeleted` and `PermissionsSynced` are available for your own code to dispatch. Each event exposes `getTimestamp()`, which is a required constructor argument: the events never read the clock themselves. The repositories pass the current instant from the [`marko/database`](/docs/packages/database/) repository (`Repository::now()`, in UTC), the same instant source used for `#[Timestamps]`.

```php
use Marko\AdminAuth\Events\PermissionsSynced;
use Psr\Clock\ClockInterface;

// $clock is an injected ClockInterface
$this->eventDispatcher->dispatch(new PermissionsSynced(
    createdCount: $created,
    totalCount: $total,
    timestamp: $clock->now(),
));
```

## API Reference

### AdminUserInterface

```php
interface AdminUserInterface extends AuthenticatableInterface
{
    public function getEmail(): string;
    public function getName(): string;
    public function setRoles(array $roles, array $permissionKeys = []): void;
    public function getRoles(): array;
    public function getPermissionKeys(): array;
    public function hasPermission(string $key): bool;
    public function hasRole(string $slug): bool;
}
```

### PermissionRegistryInterface

```php
interface PermissionRegistryInterface
{
    public function register(string $key, string $label, string $group): void;
    public function all(): array;
    public function getByGroup(string $group): array;
    public function matches(string $pattern, string $permissionKey): bool;
}
```

### RequiresPermission Attribute

```php
#[RequiresPermission(permission: 'section.action')]
```

### AdminAuthMiddleware

```php
class AdminAuthMiddleware implements MiddlewareInterface
{
    public function handle(Request $request, callable $next): Response;
}
```

Throws `Marko\Authentication\Exceptions\UnauthenticatedException` with status `401` for an unauthenticated request that wants JSON or whose guard is stateless (with the guard's `WWW-Authenticate` challenge when the guard is stateless), or `Marko\Routing\Exceptions\HttpException` with status `403` for a missing permission. An unauthenticated request on a stateful guard that doesn't want JSON gets a redirect response to `{prefix}/login`; a stateless guard never redirects.

Permission enforcement relies on the router attaching route context to the request before middleware runs (see [`marko/routing`](/docs/packages/routing/) --- `Request::withRoute()`). If no route context is present, `#[RequiresPermission]` is not evaluated and the request passes through authenticated.

### Repository Interfaces

```php
interface AdminUserRepositoryInterface extends RepositoryInterface
{
    public function findByEmail(string $email): ?AdminUser;
    public function getRolesForUser(int $userId): array;
    public function syncRoles(int $userId, array $roleIds): void;
}

interface RoleRepositoryInterface extends RepositoryInterface
{
    public function findBySlug(string $slug): ?Role;
    public function getPermissionsForRole(int $roleId): array;
    /** @return array<Permission> */
    public function getPermissionsForRoles(array $roleIds): array;
    public function syncPermissions(int $roleId, array $permissionIds): void;
    public function isSlugUnique(string $slug, ?int $excludeId = null): bool;
}

interface PermissionRepositoryInterface extends RepositoryInterface
{
    public function findByKey(string $key): ?Permission;
    public function findByGroup(string $group): array;
    public function syncFromRegistry(PermissionRegistryInterface $registry): void;
}
```

`getPermissionsForRoles()` returns the deduplicated permission set across all given role IDs in a single query. Empty input returns an empty array without issuing a query.

`syncPermissions()` replaces a role's permissions: it deletes the existing rows and inserts the new set in batches, all inside one `transaction()`. A failure part-way through rolls the whole sync back, so a role is never left half-synced. Called inside your own transaction, the sync runs in a savepoint. If it fails, only the sync's changes are undone, and you can catch the exception and still commit the rest of your transaction.

### AdminAuthConfigInterface

```php
interface AdminAuthConfigInterface
{
    public function getGuardName(): string;
    public function getSuperAdminRoleSlug(): string;
}
```
