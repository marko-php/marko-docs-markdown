---
title: marko/admin
description: Admin contracts and section registry -- defines the structure for admin sections, menu items, and dashboard widgets so any module can contribute to the admin panel.
---

Admin contracts and section registry --- defines the structure for admin sections, menu items, and dashboard widgets so any module can contribute to the admin panel. Modules declare admin sections with `#[AdminSection]` attributes, each containing menu items with permission-based visibility. At boot the package finds every `#[AdminSection]` class and registers it in the shared `AdminSectionRegistry`, which serves the sections sorted by priority. This is an interface/contracts package --- install `marko/admin-panel` or `marko/admin-api` for the actual admin UI.

## Installation

```bash
composer require marko/admin
```

## Usage

### Registering an Admin Section

Create a class that implements `AdminSectionInterface` and mark it with `#[AdminSection]`:

```php
use Marko\Admin\Attributes\AdminSection;
use Marko\Admin\Contracts\AdminSectionInterface;
use Marko\Admin\Contracts\MenuItemInterface;
use Marko\Admin\MenuItem;

#[AdminSection(
    id: 'catalog',
    label: 'Catalog',
    icon: 'box',
    sortOrder: 20,
)]
class CatalogSection implements AdminSectionInterface
{
    public function getId(): string
    {
        return 'catalog';
    }

    public function getLabel(): string
    {
        return 'Catalog';
    }

    public function getIcon(): string
    {
        return 'box';
    }

    public function getSortOrder(): int
    {
        return 20;
    }

    /**
     * @return array<MenuItemInterface>
     */
    public function getMenuItems(): array
    {
        return [
            new MenuItem(
                id: 'products',
                label: 'Products',
                url: '/admin/catalog/products',
                icon: 'package',
                sortOrder: 10,
                permission: 'catalog.products.view',
            ),
            new MenuItem(
                id: 'categories',
                label: 'Categories',
                url: '/admin/catalog/categories',
                icon: 'folder',
                sortOrder: 20,
                permission: 'catalog.categories.view',
            ),
        ];
    }
}
```

### How Sections Are Discovered

You don't register a section yourself. Put the class anywhere under a module's `src/` directory (in `vendor/`, `modules/` or `app/`), and marko/admin's boot callback finds it:

1. Every enabled module's `src/` is scanned for classes marked with `#[AdminSection]`. Every class in a file is checked, and the attribute is found however it's written: imported, fully qualified (`#[\Marko\Admin\Attributes\AdminSection(...)]`), under an alias, or grouped with other attributes.
2. Each class is parsed into an `AdminSectionDefinition`.
3. Each class is resolved through the container, so a section can inject dependencies in its constructor, and registered in `AdminSectionRegistryInterface`.

Boot does not touch the database. Each section is built once per boot, which means once per worker under a long-running server such as RoadRunner. A section must not keep request or user data in its properties.

Boot fails loudly with an `AdminException` when:

- two classes declare the same section id (the message names both classes). A section you also register by hand with `register()` fails the same way, so remove the manual call.
- `getId()` returns something other than the `#[AdminSection]` id.
- a class marked with `#[AdminSection]` does not implement `AdminSectionInterface`.

In production, `marko discovery:cache` stores the section list in the discovery cache under `admin_sections`, so a cached boot registers sections and their permissions without scanning or reflection. After adding or changing an `#[AdminSection]` or `#[AdminPermission]` class, recompile the cache with `marko discovery:cache`. If the cache names a class that no longer exists, boot fails and tells you to recompile.

### Declaring Permissions

Use `#[AdminPermission]` to declare permissions that your section requires. The attribute is repeatable, so you can stack multiple permissions on a single class:

```php
use Marko\Admin\Attributes\AdminPermission;
use Marko\Admin\Attributes\AdminSection;
use Marko\Admin\Contracts\AdminSectionInterface;

#[AdminSection(id: 'catalog', label: 'Catalog')]
#[AdminPermission(id: 'catalog.products.view', label: 'View Products')]
#[AdminPermission(id: 'catalog.products.edit', label: 'Edit Products')]
class CatalogSection implements AdminSectionInterface
{
    // ...
}
```

marko/admin only reads these attributes; permissions belong to [marko/admin-auth](/docs/packages/admin-auth/). When marko/admin-auth is installed, its boot callback registers each `#[AdminPermission]` in `PermissionRegistryInterface`, with the first segment of the key as the group. Run `marko admin-auth:permissions:sync` to write them to the database.

### Querying Sections

`AdminSectionRegistryInterface` is a shared singleton, so every class that injects it sees the sections registered at boot. Inject it to access all registered sections:

```php
use Marko\Admin\Contracts\AdminSectionInterface;
use Marko\Admin\Contracts\AdminSectionRegistryInterface;

readonly class NavigationBuilder
{
    public function __construct(
        private AdminSectionRegistryInterface $adminSectionRegistry,
    ) {}

    public function buildMenu(): array
    {
        $sections = $this->adminSectionRegistry->all(); // sorted by sortOrder

        return array_map(
            fn (AdminSectionInterface $section) => [
                'label' => $section->getLabel(),
                'items' => $section->getMenuItems(),
            ],
            $sections,
        );
    }
}
```

### Creating Dashboard Widgets

Implement `DashboardWidgetInterface` to add widgets to the admin dashboard:

```php
use Marko\Admin\Contracts\DashboardWidgetInterface;

class RecentOrdersWidget implements DashboardWidgetInterface
{
    public function getId(): string
    {
        return 'recent-orders';
    }

    public function getLabel(): string
    {
        return 'Recent Orders';
    }

    public function getSortOrder(): int
    {
        return 10;
    }

    public function render(): string
    {
        return '<div>Order list here</div>';
    }
}
```

### Admin Configuration

The `AdminConfigInterface` provides access to admin panel settings such as the route prefix and display name. It reads from [marko/config](/docs/packages/config/) under the `admin` namespace:

```php title="config/admin.php"
return [
    'route_prefix' => '/admin',
    'name' => 'My Admin Panel',
];
```

```php
use Marko\Admin\Config\AdminConfigInterface;

readonly class AdminRouter
{
    public function __construct(
        private AdminConfigInterface $adminConfig,
    ) {}

    public function getBaseUrl(): string
    {
        return $this->adminConfig->getRoutePrefix(); // e.g. '/admin'
    }

    public function getPanelName(): string
    {
        return $this->adminConfig->getName(); // e.g. 'My Admin Panel'
    }
}
```

The route prefix is validated on access --- it must start with `/` or an `InvalidAdminConfigException` is thrown.

## API Reference

### AdminSectionInterface

```php
interface AdminSectionInterface
{
    public function getId(): string;
    public function getLabel(): string;
    public function getIcon(): string;
    public function getSortOrder(): int;
    public function getMenuItems(): array;
}
```

### AdminSectionRegistryInterface

```php
interface AdminSectionRegistryInterface
{
    public function register(AdminSectionInterface $section): void;
    public function all(): array;
    public function get(string $id): AdminSectionInterface;
}
```

Calling `register()` with a duplicate section id throws `AdminException`. Calling `get()` with an unknown id also throws `AdminException`.

### MenuItemInterface

```php
interface MenuItemInterface
{
    public function getId(): string;
    public function getLabel(): string;
    public function getUrl(): string;
    public function getIcon(): string;
    public function getSortOrder(): int;
    public function getPermission(): string;
}
```

A concrete `MenuItem` class is provided with all properties accepted as constructor parameters. The `icon`, `sortOrder`, and `permission` parameters are optional (defaulting to `''`, `0`, and `''` respectively).

### DashboardWidgetInterface

```php
interface DashboardWidgetInterface
{
    public function getId(): string;
    public function getLabel(): string;
    public function getSortOrder(): int;
    public function render(): string;
}
```

### AdminConfigInterface

```php
interface AdminConfigInterface
{
    public function getRoutePrefix(): string;
    public function getName(): string;
}
```

### Attributes

| Attribute | Target | Parameters |
|-----------|--------|------------|
| `#[AdminSection]` | Class | `id`, `label`, `icon` (optional), `sortOrder` (optional) |
| `#[AdminPermission]` | Class (repeatable) | `id`, `label` (optional) |

### AdminSectionDiscovery

`AdminSectionDiscovery` finds section classes and reads their metadata:

```php
public function discoverAll(array $modules): array;
public function discoverInModule(ModuleManifest $manifest): array;
public function parseAdminSectionClass(string $className): AdminSectionDefinition;
```

`discoverAll()` parses every section across the given modules, in module order, into `AdminSectionDefinition`s. Two sections with the same id throw `AdminException::duplicateSection()` naming both classes.

`discoverInModule()` returns the names of every class in the module's `src/` directory that is marked with `#[AdminSection]`. A cheap text match picks candidate files: a file must contain an attribute and mention `AdminSection`. Every class in a candidate file is then loaded and checked with reflection. A file that only mentions `#[AdminSection` in a comment, or uses a longer attribute name such as `#[AdminSectionWidget]`, is skipped. A class that really carries the attribute is always reported, even when it is invalid, so `parseAdminSectionClass()` can reject it loudly.

`parseAdminSectionClass()` turns a section class into an `AdminSectionDefinition` with its `#[AdminPermission]` entries. It checks the attribute first, then the interface.

### DiscoveredAdminSections

`DiscoveredAdminSections::all()` returns the section definitions for the current boot. On a cached boot it reads the `admin_sections` cache section; otherwise it scans the enabled modules. The work runs once and the result is kept. It is a shared singleton: marko/admin's boot callback uses it to register the sections, and marko/admin-auth's uses it to register their permissions.

### AdminSectionCacheContributor

The discovery cache contributor declared in `module.php` under `discovery`, with the key `admin_sections`. `compile()` stores each definition as plain values: class name, id, label, icon, sort order and permissions. `hydrate()` rebuilds the definitions and throws `DiscoveryCacheException` for a malformed record.

### Exceptions

Section registry and discovery errors are thrown as `AdminException`, each with a message, context and suggestion:

| Factory | Thrown when |
|---------|-------------|
| `AdminException::duplicateSection()` | Two sections share an id, in `discoverAll()` or `AdminSectionRegistry::register()`. The message names both classes |
| `AdminException::sectionIdMismatch()` | At boot, a section's `getId()` differs from its `#[AdminSection]` id |
| `AdminException::sectionClassNotFound()` | At boot, the discovery cache names a section class that no longer exists |
| `AdminException::sectionNotFound()` | `AdminSectionRegistry::get()` is called with an unknown id |
| `AdminException::missingSectionAttribute()` | `parseAdminSectionClass()` is given a class that is not marked with `#[AdminSection]` |
| `AdminException::sectionMustImplementInterface()` | `parseAdminSectionClass()` is given a class marked with `#[AdminSection]` that does not implement `AdminSectionInterface`, or boot resolves a cached section class that does not |

`InvalidAdminConfigException` is thrown by `AdminConfig` when the route prefix does not start with `/`.
