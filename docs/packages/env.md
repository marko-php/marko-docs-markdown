---
title: marko/env
description: Environment variable loading — reads .env files and provides the env() helper with automatic type coercion.
---

Environment variable loading — reads `.env` files and provides the `env()` helper with automatic type coercion. Env loads variables from a `.env` file into `$_ENV` and `putenv()`, with system environment variables taking precedence. The `env()` helper function retrieves values with type coercion for common patterns (`true`, `false`, `null`, `empty`). No external dependencies — works with any PHP application.

## Installation

```bash
composer require marko/env
```

## Usage

### Loading Environment Variables

Load from a `.env` file at application bootstrap:

```php
use Marko\Env\EnvLoader;

$envLoader = new EnvLoader();
$envLoader->load(__DIR__);
```

The `.env` file is optional. Real environment variables always take precedence over `.env` values, so production deployments can override anything from the container, PaaS, or web server.

### Real Environment Variables Are Mirrored Into `$_ENV`

Before reading `.env`, the loader copies every real environment variable (everything `getenv()` returns) into `$_ENV`, without overwriting entries that are already there. This happens even when no `.env` file exists.

PHP only fills `$_ENV` itself when the `variables_order` ini setting contains `E`, and the `php.ini-production` default (`GPCS`) does not. Without mirroring, a config file such as `'host' => $_ENV['DB_HOST'] ?? 'localhost'` would silently fall back to its default in a typical container deployment. With mirroring, `$_ENV` and `getenv()` agree no matter how PHP is configured, so config files can safely read `$_ENV`, which is where [`Marko\Config\Env`](/docs/packages/config/#environment-variables) looks first.

### The `env()` Helper

:::note
In config files, read environment variables with [`Marko\Config\Env`](/docs/packages/config/#environment-variables) instead. `Env::int()`, `Env::bool()` and the other typed readers throw on a value they can't parse, while `env()` coerces only the strings in the table below and returns everything else unchanged: `DEBUGBAR_ENABLED=off` reaches the config as the string `'off'`. The config files Marko ships no longer call `env()`. `Env` also doesn't treat `null` or `empty` as special words: unset the variable or leave it empty to use the default.
:::

Retrieve environment variables with automatic type coercion:

```php
// Simple retrieval
$dbHost = env('DB_HOST', 'localhost');

// Type coercion
$debug = env('APP_DEBUG');    // 'true' -> true, 'false' -> false
$value = env('NULLABLE_VAR'); // 'null' -> null
$empty = env('EMPTY_VAR');    // 'empty' -> ''
```

### .env File Format

```
APP_NAME=Marko
APP_DEBUG=true
DB_HOST=localhost
DB_PORT=3306

# Comments start with #
SECRET_KEY="quoted values supported"
ANOTHER_KEY='single quotes too'
```

### Using in Config Files

Environment variables should only be referenced in [config](/docs/packages/config/) files, not in application code:

```php title="config/database.php"
<?php

declare(strict_types=1);

use Marko\Config\Env;

return [
    'host' => Env::string('DB_HOST', 'localhost'),
    'port' => Env::int('DB_PORT', 3306, min: 1, max: 65535),
    'name' => Env::string('DB_NAME', 'marko'),
];
```

`Env` lives in [marko/config](/docs/packages/config/#environment-variables), which documents every reader, the accepted values, and the rule that an unset or empty variable returns the default.

Application code then reads config values:

```php
$host = $this->config->getString('database.host');
```

## Type Coercion

| String Value | Coerced To |
|--------------|------------|
| `'true'`, `'(true)'` | `true` |
| `'false'`, `'(false)'` | `false` |
| `'null'`, `'(null)'` | `null` |
| `'empty'`, `'(empty)'` | `''` |
| Everything else | Unchanged string |

## API Reference

### EnvLoader

```php
public function load(string $path): void;
```

Mirrors real environment variables (from `getenv()`) into `$_ENV` without overwriting existing entries, then loads environment variables from a `.env` file in the given directory, if one exists. Skips comments, blank lines, and lines without `=`. Removes surrounding quotes (single or double) from values. Does not overwrite existing system environment variables — real environment variables win over `.env` values.

### env()

```php
function env(string $key, mixed $default = null): mixed;
```

Retrieves an environment variable by name. Checks `$_ENV` first, then falls back to `getenv()`. Returns the `$default` if the variable is not set. Applies type coercion for common string representations (see table above).
