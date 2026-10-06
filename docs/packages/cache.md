---
title: marko/cache
description: Interfaces for caching — defines how data is stored, retrieved, and expired, not how the backend works.
---

Interfaces for caching --- defines how data is stored, retrieved, and expired, not how the backend works. Cache provides the contracts and shared infrastructure for Marko's caching system. Type-hint against `CacheInterface` in your modules and let the installed driver handle the backend. Includes CLI commands for cache management, a `CacheItem` value object with metadata, and key validation.

**This package defines contracts only.** Install a driver for implementation:

- `marko/cache-array` --- In-memory (development/testing)
- `marko/cache-file` --- File-based (default)
- `marko/cache-redis` --- Redis (production)

## Installation

```bash
composer require marko/cache
```

Note: You typically install a driver package (like `marko/cache-file`) which requires this automatically.

## Usage

### Type-Hinting the Cache

Inject `CacheInterface` wherever you need caching:

```php
use Marko\Cache\Contracts\CacheInterface;

class ProductService
{
    public function __construct(
        private CacheInterface $cache,
    ) {}

    public function getProduct(
        int $id,
    ): Product {
        $key = "product.$id";

        if ($this->cache->has($key)) {
            return $this->cache->get($key);
        }

        $product = $this->repository->find($id);
        $this->cache->set($key, $product, ttl: 3600);

        return $product;
    }
}
```

### Cache Item Metadata

Use `getItem()` when you need expiration info alongside the value:

```php
use Marko\Cache\Contracts\CacheInterface;

$item = $this->cache->getItem('product.42');

if ($item->isHit()) {
    $value = $item->get();
    $expiresAt = $item->expiresAt();
}
```

The `CacheItem` value object also provides static factory methods for driver implementations:

```php
use Marko\Cache\CacheItem;
use DateTimeImmutable;

// Create a cache hit
$item = CacheItem::hit('product.42', $value, new DateTimeImmutable('+1 hour'));

// Create a cache miss
$item = CacheItem::miss('product.42');
```

### Batch Operations

Store and retrieve multiple values at once:

```php
use Marko\Cache\Contracts\CacheInterface;

$this->cache->setMultiple([
    'user.1' => $user1,
    'user.2' => $user2,
], ttl: 600);

$users = $this->cache->getMultiple(['user.1', 'user.2']);
```

### Key Validation

Cache keys are validated automatically. Keys cannot be empty or contain reserved characters: `/ \ : * ? " < > | { }`. Invalid keys throw an `InvalidKeyException` with a helpful message:

```php
use Marko\Cache\Exceptions\InvalidKeyException;

// Check key validity without throwing
$valid = InvalidKeyException::isValidKey('my.cache.key'); // true
$valid = InvalidKeyException::isValidKey('invalid/key');  // false
```

### Configuration

The `CacheConfig` class provides typed access to cache configuration values:

```php
use Marko\Cache\Config\CacheConfig;

class MyService
{
    public function __construct(
        private CacheConfig $cacheConfig,
    ) {}

    public function setup(): void
    {
        $driver = $this->cacheConfig->driver();
        $path = $this->cacheConfig->path();
        $defaultTtl = $this->cacheConfig->defaultTtl();
    }
}
```

### Expiry and the Clock

The bundled drivers ([`marko/cache-array`](/docs/packages/cache-array/), [`marko/cache-file`](/docs/packages/cache-file/) and [`marko/cache-redis`](/docs/packages/cache-redis/)) read the current time through the PSR-20 `ClockInterface` from [`marko/clock`](/docs/packages/clock/) instead of calling `time()`. TTLs, expiry checks and `expiresAt()` all follow that clock. To test expiry without `sleep()`, construct a driver with a [`FakeClock`](/docs/packages/testing/#fakeclock) and move it:

```php
use Marko\Cache\Config\CacheConfig;
use Marko\Cache\Memory\Driver\ArrayCacheDriver;
use Marko\Testing\Fake\FakeClock;
use Marko\Testing\Fake\FakeConfigRepository;

it('expires a cached value after its ttl', function (): void {
    $clock = new FakeClock('2026-01-01 12:00:00 UTC');
    $cache = new ArrayCacheDriver(
        new CacheConfig(new FakeConfigRepository(['cache.default_ttl' => 3600])),
        $clock,
    );
    $cache->set('product.42', 'cached', ttl: 60);

    $clock->travel('+60 seconds');
    expect($cache->get('product.42'))->toBe('cached');

    $clock->travel('+1 second');
    expect($cache->get('product.42'))->toBeNull();
});
```

Redis enforces TTLs on the server, so with `marko/cache-redis` the clock only anchors the reported `expiresAt()`. Moving a `FakeClock` does not expire Redis keys.

## CLI Commands

| Command | Description |
|---------|-------------|
| `marko cache:clear` | Clear all cached items |
| `marko cache:status` | Show cache driver and statistics |

## API Reference

### CacheInterface

```php
use Marko\Cache\Contracts\CacheInterface;
use Marko\Cache\Contracts\CacheItemInterface;

public function get(string $key, mixed $default = null): mixed;
public function set(string $key, mixed $value, ?int $ttl = null): bool;
public function has(string $key): bool;
public function delete(string $key): bool;
public function clear(): bool;
public function getItem(string $key): CacheItemInterface;
public function getMultiple(array $keys, mixed $default = null): iterable;
public function setMultiple(array $values, ?int $ttl = null): bool;
public function deleteMultiple(array $keys): bool;
public function increment(string $key, int $ttl): int;
```

All methods that accept keys throw `InvalidKeyException` for empty or invalid keys.

`increment()` atomically increments the integer stored at `$key` and returns the new value. If the key does not exist it is created with value `1` and the TTL is applied at that moment. On subsequent increments the TTL is **not** reset --- resetting on every call would turn a fixed rate-limit window into a never-closing window. The counter is a plain integer: on every driver, `get()`, `getItem()` and `getMultiple()` return an incremented key as an `int`.

### CacheItemInterface

```php
use Marko\Cache\Contracts\CacheItemInterface;
use DateTimeInterface;

public function getKey(): string;
public function get(): mixed;
public function isHit(): bool;
public function expiresAt(): ?DateTimeInterface;
```

### CacheConfig

```php
use Marko\Cache\Config\CacheConfig;

public function driver(): string;
public function path(): string;
public function defaultTtl(): int;
```

### Exceptions

| Exception | Description |
|-----------|-------------|
| `CacheException` | Base exception for all cache errors --- includes `getContext()` and `getSuggestion()` methods |
| `InvalidKeyException` | Thrown when a cache key is empty or contains reserved characters |
| `ItemNotFoundException` | Thrown when a requested cache item does not exist |
| `TamperedCacheValueException` | Thrown when `encryption.key` is empty, so cache values can't be signed or verified. The Redis driver also throws it when a stored value's HMAC does not verify |

### CacheValueSigner

`Marko\Cache\Signer\CacheValueSigner` signs serialized cache payloads with HMAC-SHA256. Drivers that persist values outside the PHP process (`marko/cache-file`, `marko/cache-redis`) use it so `unserialize()` only ever sees bytes the application wrote itself. The envelope format is `{64-char-hex-hmac}.{serialized-payload}`.

- **Derived key.** The HMAC key is a subkey derived from `encryption.key` ([`marko/encryption`](/docs/packages/encryption/)) with `hash_hkdf('sha256', $key, 32, 'marko-cache-signer')`. The key that encrypts data is never reused for signing, and the cache subkey differs from the [queue envelope's](/docs/packages/queue/#payload-envelope-format).
- **Bound to the storage key.** Every method takes a `$context` string, and the MAC covers it along with the payload. Drivers pass the key the entry is stored under (the prefixed Redis key, or the cache key for files), so a validly signed value copied to another key does not verify.

```php
use Marko\Cache\Signer\CacheValueSigner;

public function wrap(string $serialized, string $context): string;            // sign a payload for $context
public function unwrap(string $envelope, string $context): ?string;           // null when unsigned, malformed, tampered or signed for another context
public function verifyAndUnwrap(string $envelope, string $context): string;   // throws TamperedCacheValueException instead of returning null
```

Every method throws `TamperedCacheValueException::emptySigningKey()` when `encryption.key` is empty.

Entries signed by a release before HKDF subkeys and key binding no longer verify. The file driver treats them as misses and deletes them. The Redis driver throws `TamperedCacheValueException` for them, so run `marko cache:clear` when you deploy the upgrade.
