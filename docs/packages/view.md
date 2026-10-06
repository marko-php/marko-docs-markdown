---
title: marko/view
description: View and template rendering interface — defines how controllers render templates, not which engine is used.
---

View and template rendering interface --- defines how controllers render templates, not which engine is used. View provides `ViewInterface` for rendering templates into HTTP responses and `TemplateResolverInterface` for resolving template names to file paths. Templates use a `module::path` naming convention so modules own their views. This is an interface package; install a driver like `marko/view-latte` for the actual rendering engine.

## Installation

```bash
composer require marko/view
```

Note: You also need a view driver. Install `marko/view-latte` for Latte template support or `marko/view-twig` for Twig template support.

## Usage

### Rendering Templates

Inject `ViewInterface` in your controller and call `render()`:

```php
use Marko\Routing\Attributes\Get;
use Marko\Routing\Http\Response;
use Marko\View\ViewInterface;

readonly class PostController
{
    public function __construct(
        private ViewInterface $view,
    ) {}

    #[Get('/posts/{id}')]
    public function show(
        int $id,
    ): Response {
        return $this->view->render('blog::post/show', [
            'post' => $this->findPost($id),
        ]);
    }
}
```

`render()` returns a `Response` with the rendered HTML. Use `renderToString()` when you need the raw HTML:

```php
$html = $this->view->renderToString('blog::post/show', [
    'post' => $post,
]);
```

### Template Naming Convention

Templates follow the `module::path` pattern:

- `blog::post/show` --- resolves to the `blog` module's `resources/views/post/show` template
- `admin-panel::dashboard/index` --- resolves to the `admin-panel` module's dashboard view

The file extension is configured by the installed driver (e.g., `.latte` for `marko/view-latte`, `.twig` for `marko/view-twig`).

Template names are validated strictly before any file is touched. The path is one or more `/`-separated segments made only of `A-Z`, `a-z`, `0-9`, `_`, `-` and `.`, with an optional `module::` prefix. Names containing `..` or `.` segments, a leading `/`, empty segments, backslashes, NUL bytes or any other character throw `InvalidTemplateException`. After a template file is found, its real path must also sit inside the module's `resources/views/` directory, so a symlink pointing outside it is rejected as well.

Never build a template name directly from request input. Map user input to a fixed set of template names instead:

```php
$page = match ($request->query('page')) {
    'about' => 'pages::about',
    'contact' => 'pages::contact',
    default => 'pages::not-found',
};
```

### Template File Location

Place templates in your module's `resources/views/` directory:

```
mymodule/
  resources/
    views/
      post/
        index.{ext}
        show.{ext}
      layout.{ext}
```

The file extension depends on the installed driver (`.latte` for `marko/view-latte`, `.twig` for `marko/view-twig`).

### Resolving Template Paths

`TemplateResolverInterface` converts template names to absolute file paths:

```php
use Marko\View\TemplateResolverInterface;

readonly class TemplateFinder
{
    public function __construct(
        private TemplateResolverInterface $templateResolver,
    ) {}

    public function locate(
        string $template,
    ): string {
        return $this->templateResolver->resolve($template);
        // Returns: /path/to/blog/resources/views/post/show.latte (or .twig, etc.)
    }
}
```

If a template is not found, `TemplateNotFoundException` is thrown with all searched paths listed.

### Configuration

The `ViewConfig` class provides typed access to view configuration values:

```php
use Marko\View\ViewConfig;

class MyService
{
    public function __construct(
        private ViewConfig $viewConfig,
    ) {}

    public function setup(): void
    {
        $cacheDir = $this->viewConfig->cacheDirectory();
        $autoRefresh = $this->viewConfig->autoRefresh();
    }
}
```

### Compiled-Template Cache Directory

Template engines compile views to PHP files and `include` them, so the cache directory is code-execution sensitive. `view.cache_directory` defaults to `storage/views`:

```php title="config/view.php"
return [
    'cache_directory' => 'storage/views', // relative to the project root
    'auto_refresh' => true,
];
```

Drivers pass the configured value through `CacheDirectoryGuard::prepare()` before handing it to the engine. The guard:

- resolves a relative path against the project root (`ProjectPaths::$base`), never the working directory --- FPM, CGI and mod_php `chdir` into `public/`
- creates a missing directory with mode `0700`
- throws `InsecureCacheDirectoryException` when the directory is world-writable or not owned by the user PHP runs as, so another local user cannot plant compiled PHP (the classic pre-created `/tmp/views` attack)

The ownership check requires the `posix` extension; without it only the world-writable check runs. Absolute paths are still allowed, but they must pass the same checks, so a shared `/tmp/views` is refused.

## Customization

Replace the template resolver via [Preferences](/docs/packages/core/) to customize how templates are located:

```php
use Marko\Core\Attributes\Preference;
use Marko\View\ModuleTemplateResolver;

#[Preference(replaces: ModuleTemplateResolver::class)]
class ThemeTemplateResolver extends ModuleTemplateResolver
{
    public function resolve(
        string $template,
    ): string {
        // Check theme directory first, then fall back
        return parent::resolve($template);
    }
}
```

## API Reference

### ViewInterface

```php
use Marko\Routing\Http\Response;
use Marko\View\ViewInterface;

public function render(string $template, array $data = []): Response;
public function renderToString(string $template, array $data = []): string;
```

### TemplateResolverInterface

```php
use Marko\View\TemplateResolverInterface;

public function resolve(string $template): string;
public function getSearchedPaths(string $template): array;
```

### ViewConfig

```php
use Marko\View\ViewConfig;

public function cacheDirectory(): string;
public function autoRefresh(): bool;
```

### CacheDirectoryGuard

```php
use Marko\View\CacheDirectoryGuard;

public function prepare(string $directory): string;
```

Returns the resolved absolute path. Throws `InsecureCacheDirectoryException` when the directory cannot be created, is a file, is world-writable, or is owned by another user.

### Exceptions

| Exception | Description |
|-----------|-------------|
| `ViewException` | Base exception for all view errors --- extends `MarkoException` |
| `TemplateNotFoundException` | Thrown when a template cannot be found --- includes all searched paths |
| `InvalidTemplateException` | Thrown when a template name is unsafe (path traversal, absolute path, NUL byte, backslash, disallowed characters) or resolves outside its module's `resources/views/` directory |
| `InsecureCacheDirectoryException` | Thrown when the compiled-template cache directory cannot be created, is not a directory, is world-writable, or is owned by another user |
| `NoDriverException` | Thrown when no view driver is installed --- suggests `composer require marko/view-latte` or `composer require marko/view-twig` |

## Related Packages

- [`marko/view-latte`](/docs/packages/view-latte/) --- Latte templating driver
- [`marko/view-twig`](/docs/packages/view-twig/) --- Twig templating driver
