---
title: marko/env
description: Environment variable loading — reads a .env file into $_ENV at boot so config files can read it with Marko\Config\Env.
---

Environment variable loading — reads a `.env` file into `$_ENV` and `putenv()` at boot, with real environment variables taking precedence. Config files then read those values with [`Marko\Config\Env`](/docs/packages/config/#environment-variables). No external dependencies — works with any PHP application.

## Installation

```bash
composer require marko/env
```

The [skeleton](/docs/packages/skeleton/) requires `marko/env`, so a new app already has it. Framework packages don't require it: loading a `.env` file is the app's choice.

## Usage

### Loading Environment Variables

When `marko/env` is installed, [`marko/core`](/docs/packages/core/) loads the `.env` file in the project root at boot, before any config file runs. Outside a Marko app, load it yourself at bootstrap:

```php
use Marko\Env\EnvLoader;

$envLoader = new EnvLoader();
$envLoader->load(__DIR__);
```

The `.env` file is optional. Real environment variables always take precedence over `.env` values, so production deployments can override anything from the container, PaaS, or web server.

### Real Environment Variables Are Mirrored Into `$_ENV`

Before reading `.env`, the loader copies every real environment variable (everything `getenv()` returns) into `$_ENV`, without overwriting entries that are already there. This happens even when no `.env` file exists.

PHP only fills `$_ENV` itself when the `variables_order` ini setting contains `E`, and the `php.ini-production` default (`GPCS`) does not. Without mirroring, a config file such as `'host' => $_ENV['DB_HOST'] ?? 'localhost'` would silently fall back to its default in a typical container deployment. With mirroring, `$_ENV` and `getenv()` agree no matter how PHP is configured, so config files can safely read `$_ENV`, which is where [`Marko\Config\Env`](/docs/packages/config/#environment-variables) looks first.

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

### Reading Values in Config Files

Environment variables should only be referenced in [config](/docs/packages/config/) files, not in application code. Read them with `Marko\Config\Env`:

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

## Removed in 0.9.0: the `env()` Helper

:::caution
Marko 0.9.0 removed the global `env()` function, which 0.8.x deprecated. A config file that still calls it stops the app at boot with:

```text
Call to undefined function env()
```

Switch each call to `Marko\Config\Env` before upgrading, using the table below. To upgrade in steps, stay on 0.8.x until your config files are migrated: there, every `env()` call emits an `E_USER_DEPRECATED` notice that names the variable and its `Env` replacement.
:::

`env()` coerced only a few strings and returned everything else unchanged, so a typo never failed: `APP_DEBUG=off` reached the config as the string `'off'`, which is truthy. `Env::bool()` reads `off` as `false` and throws on a value such as `ture`. `marko/env` now ships only the `.env` loader.

### Before and After

The same config file, written with `env()` and with `Env`:

```php title="config/mercure.php (before)"
<?php

declare(strict_types=1);

return [
    'hub_url' => env('MERCURE_URL', 'http://localhost/.well-known/mercure'),
    'subscriber_jwt_ttl' => (int) env('MERCURE_SUBSCRIBER_JWT_TTL', 3600),
    'cookie_secure' => env('MERCURE_COOKIE_SECURE', true),
];
```

```php title="config/mercure.php (after)"
<?php

declare(strict_types=1);

use Marko\Config\Env;

return [
    'hub_url' => Env::string('MERCURE_URL', 'http://localhost/.well-known/mercure'),
    'subscriber_jwt_ttl' => Env::int('MERCURE_SUBSCRIBER_JWT_TTL', 3600, min: 0),
    'cookie_secure' => Env::bool('MERCURE_COOKIE_SECURE', true),
];
```

What each version does with the same `.env` values:

| `.env` value | `env()` version | `Env` version |
|---|---|---|
| `MERCURE_SUBSCRIBER_JWT_TTL=7200` | `7200` | `7200` |
| `MERCURE_SUBSCRIBER_JWT_TTL=1h` | `(int) '1h'` is `1`, so tokens expire after one second | Throws `ConfigException`: `Environment variable "MERCURE_SUBSCRIBER_JWT_TTL" must be an integer`, `Got "1h"` |
| `MERCURE_SUBSCRIBER_JWT_TTL=-5` | `-5` | Throws, because of `min: 0` |
| `MERCURE_COOKIE_SECURE=off` | The string `'off'`, which is truthy, so the cookie stays secure when you meant to turn it off | `false` |
| `MERCURE_COOKIE_SECURE=ture` | The string `'ture'`, accepted without complaint | Throws: `must be a boolean`, `Got "ture"`, listing the accepted values |
| `MERCURE_COOKIE_SECURE=` (empty) | `''`, which is falsy | `true`, the default |
| Variable not set | The default | The default |

`Env` errors are thrown while the config file loads, so a bad value stops the app at boot, and the message names the config file that read it.

Replace each call with the `Env` method for the type the config value needs:

| `env()` call | `env()` behavior | Replace with |
|---|---|---|
| `env('APP_DEBUG', false)` | `'true'`, `'(true)'` → `true`; `'false'`, `'(false)'` → `false`; any other string returned unchanged | `Env::bool('APP_DEBUG', false)`. Accepts `true`, `false`, `1`, `0`, `yes`, `no`, `on`, `off`; `(true)` and `(false)` throw, so write `true` or `false` |
| `env('MAIL_FROM')` with `MAIL_FROM=null` | `'null'`, `'(null)'` → `null` | `Env::nullableString('MAIL_FROM')`, and unset the variable or leave it empty (`MAIL_FROM=`). `Env` doesn't treat the word `null` as special |
| `env('PREFIX', 'app_')` with `PREFIX=empty` | `'empty'`, `'(empty)'` → `''` | Set the empty string in the config file (`'prefix' => ''`). `Env` treats an empty value as unset and returns the default, and doesn't treat the word `empty` as special |
| `env('DB_HOST', 'localhost')` | Any other string, unchanged | `Env::string('DB_HOST', 'localhost')` |
| `env('DB_PORT', 3306)` | The string `'3306'`, not an `int` | `Env::int('DB_PORT', 3306)` |

`env()` also differed on an empty value: `KEY=` returned `''` from `env()` and returns the default from `Env`.

Search your config files for `env(` rather than relying on the error. If an upgraded app boots without it but still has `env()` calls, another installed library defines a global `env()` (Laravel's `illuminate/support` helpers are the common one), and those calls now run that library's function, with its own coercion rules. Migrate them to `Env` all the same. `Env` is a class, so it has no such conflict.

## API Reference

### EnvLoader

```php
public function load(string $path): void;
```

Mirrors real environment variables (from `getenv()`) into `$_ENV` without overwriting existing entries, then loads environment variables from a `.env` file in the given directory, if one exists. Skips comments, blank lines, and lines without `=`. Removes surrounding quotes (single or double) from values. Does not overwrite existing system environment variables — real environment variables win over `.env` values.
