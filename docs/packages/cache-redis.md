---
title: marko/cache-redis
description: Redis cache driver — fast, persistent caching backed by Redis for production workloads.
---

Redis cache driver --- fast, persistent caching backed by Redis for production workloads. Stores HMAC-signed, serialized data in Redis with automatic TTL expiration. Values are tamper-evident: reads verify the signature before deserializing, so modified entries are rejected loudly. Supports key prefixing to isolate cache namespaces, configurable host/port/database, and optional authentication. Uses Predis as the Redis client library.

Implements `CacheInterface` from [`marko/cache`](/docs/packages/cache/).

## Installation

```bash
composer require marko/cache-redis
```

This automatically installs `marko/cache`, `predis/predis`, and [`marko/encryption`](/docs/packages/encryption/). A non-empty `encryption.key` is required; reads and writes throw `TamperedCacheValueException` if the key is empty or a stored value's HMAC does not verify.

```php title="config/encryption.php"
return [
    'key' => $_ENV['APP_KEY'] ?? '',
];
```

## Configuration

Set the cache driver to `redis` in your config:

```php title="config/cache.php"
return [
    'driver' => 'redis',
    'default_ttl' => 3600,
    'path' => 'storage/cache',
];
```

The package ships `config/cache-redis.php`, and its module binding builds a single shared `RedisConnection` from it. No hand-written binding is needed. Set the environment variables, or override the file in your app:

```php title="config/cache-redis.php"
return [
    'host' => $_ENV['REDIS_HOST'] ?? '127.0.0.1',
    'port' => (int) ($_ENV['REDIS_PORT'] ?? 6379),
    'password' => $_ENV['REDIS_PASSWORD'] ?? null,
    'database' => (int) ($_ENV['REDIS_CACHE_DATABASE'] ?? 0),
    'prefix' => $_ENV['CACHE_PREFIX'] ?? 'marko:cache:',
];
```

| Key | Env var | Default | Description |
|---|---|---|---|
| `host` | `REDIS_HOST` | `127.0.0.1` | Redis server host |
| `port` | `REDIS_PORT` | `6379` | Redis server port |
| `password` | `REDIS_PASSWORD` | `null` | Password for `AUTH`; `null` or empty means no authentication |
| `database` | `REDIS_CACHE_DATABASE` | `0` | Redis database index |
| `prefix` | `CACHE_PREFIX` | `marko:cache:` | Prefix added to every cache key |

The connection opens on first use. If Redis refuses it, a `RedisConnectionException` names the host and port and points back to this config file.

## Usage

Once configured, inject `CacheInterface` as usual --- the Redis driver is used automatically:

```php
use Marko\Cache\Contracts\CacheInterface;

class SessionStore
{
    public function __construct(
        private CacheInterface $cache,
    ) {}

    public function getSession(
        string $token,
    ): ?array {
        return $this->cache->get("session.$token");
    }

    public function saveSession(
        string $token,
        array $data,
    ): void {
        $this->cache->set("session.$token", $data, ttl: 1800);
    }
}
```

### When to Use

- **Production workloads** with high read/write throughput
- **Multi-server deployments** where cache must be shared
- **Session storage** and other latency-sensitive data
- **TTL-managed expiration** handled natively by Redis

### Key Prefixing

All keys are automatically prefixed (default: `marko:cache:`) to prevent collisions with other Redis data. Change it with the `prefix` key in `config/cache-redis.php` (or `CACHE_PREFIX`).

## API Reference

Implements all methods from `CacheInterface`. See [`marko/cache`](/docs/packages/cache/) for the full contract.

### Key Methods

| Method | Description |
|---|---|
| `get(string $key, mixed $default = null): mixed` | Retrieve a value, returning `$default` on miss or expiration |
| `set(string $key, mixed $value, ?int $ttl = null): bool` | Store a value with optional TTL (falls back to `default_ttl`) |
| `has(string $key): bool` | Check if a non-expired entry exists |
| `delete(string $key): bool` | Remove a single entry |
| `clear(): bool` | Remove all prefixed entries from Redis |
| `getItem(string $key): CacheItemInterface` | Get a `CacheItem` with hit/miss status and expiration metadata |
| `getMultiple(array $keys, mixed $default = null): iterable` | Retrieve multiple values at once |
| `setMultiple(array $values, ?int $ttl = null): bool` | Store multiple key-value pairs at once |
| `deleteMultiple(array $keys): bool` | Remove multiple entries at once |
| `increment(string $key, int $ttl): int` | Atomically increment an integer counter and apply its TTL in one step; an existing TTL is never reset |

### RedisConnection

| Method | Description |
|---|---|
| `__construct(string $host, int $port, ?string $password, int $database, string $prefix)` | Create a connection with host (`127.0.0.1`), port (`6379`), optional password, database index (`0`), and key prefix (`marko:cache:`) |
| `client(): ClientInterface` | Get the Predis client instance --- connected on first call; throws `RedisConnectionException` if the connection is refused |
| `disconnect(): void` | Disconnect and release the client instance |
| `isConnected(): bool` | Check whether a client instance is currently active |

### Storage Details

- Values are serialized with PHP's `serialize()`, wrapped in an HMAC-SHA256 envelope, and stored as Redis strings. Reads verify the HMAC before deserializing; tampered or corrupted entries throw `TamperedCacheValueException`.
- A `null` TTL falls back to `default_ttl` from config. A TTL greater than `0` uses Redis `SETEX` for native expiration. A TTL of `0` or less means the entry never expires.
- `increment()` runs `INCR` and `EXPIRE` together in one Lua script (`EVAL`), so a crash can never leave a counter without an expiry. The TTL is set when the key is created (count reaches `1`), and is restored if the key somehow has none. Subsequent increments do not reset it, so the window stays fixed.
- Counters are plain integers, not signed envelopes, because Redis writes them itself. `get()`, `getItem()` and `getMultiple()` return them as `int`, which is what [`marko/ratelimiter`](/docs/packages/ratelimiter/) relies on. Only values that are entirely an optional `-` followed by digits are read this way. They are never passed to `unserialize()`, and every other value must still carry a valid HMAC.
- `clear()` removes only keys matching the configured prefix --- other Redis data is not affected.
