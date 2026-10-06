---
title: marko/encryption
description: Interfaces for encryption — defines how data is encrypted and decrypted, not which cipher is used.
---

Interfaces for encryption --- defines how data is encrypted and decrypted, not which cipher is used. Encryption provides the `EncryptorInterface` contract and shared infrastructure for Marko's encryption system. Type-hint against the interface in your modules and let the installed driver handle the cryptographic implementation. Includes configuration for key management and rich exceptions for invalid keys and corrupted payloads.

**This package defines contracts only.** Install a driver for implementation:

- [`marko/encryption-openssl`](/docs/packages/encryption-openssl/) --- OpenSSL with AES-256-GCM (recommended)

## Installation

```bash
composer require marko/encryption
```

Note: You typically install a driver package (like `marko/encryption-openssl`) which requires this automatically.

## Usage

### Type-Hinting the Encryptor

Inject `EncryptorInterface` wherever you need encryption:

```php
use JsonException;
use Marko\Encryption\Contracts\EncryptorInterface;

readonly class TokenService
{
    public function __construct(
        private EncryptorInterface $encryptor,
    ) {}

    /**
     * @throws JsonException
     */
    public function issueToken(
        array $payload,
    ): string {
        $json = json_encode($payload, JSON_THROW_ON_ERROR);

        return $this->encryptor->encrypt($json);
    }

    /**
     * @throws JsonException
     */
    public function verifyToken(
        string $token,
    ): array {
        $json = $this->encryptor->decrypt($token);

        return json_decode($json, true, flags: JSON_THROW_ON_ERROR);
    }
}
```

### Encrypting Entity Columns

To store an entity property encrypted, mark it `#[Encrypted]` instead of encrypting and decrypting by hand in a repository. `marko/database` encrypts on save and decrypts on hydration using the bound `EncryptorInterface`. See [Encrypted Columns](/docs/packages/database/#encrypted-columns).

### Configuration

The `EncryptionConfig` class provides typed access to encryption configuration values:

```php
use Marko\Encryption\Config\EncryptionConfig;

class MyService
{
    public function __construct(
        private EncryptionConfig $encryptionConfig,
    ) {}

    public function setup(): void
    {
        $key = $this->encryptionConfig->key();
        $cipher = $this->encryptionConfig->cipher();
    }
}
```

Set the encryption key and cipher in your config:

```php title="config/encryption.php"
use Marko\Config\Env;

return [
    'key' => Env::string('ENCRYPTION_KEY', ''),
    'cipher' => Env::string('ENCRYPTION_CIPHER', 'aes-256-gcm'),
];
```

Generate a key with: `base64_encode(random_bytes(32))`

## API Reference

### EncryptorInterface

```php
use Marko\Encryption\Contracts\EncryptorInterface;

public function encrypt(string $value): string;
public function decrypt(string $encrypted): string;
```

`encrypt()` throws `EncryptionException` on failure. `decrypt()` throws `DecryptionException` for invalid payloads, wrong keys, or tampered data.

### EncryptionConfig

```php
use Marko\Encryption\Config\EncryptionConfig;

public function key(): string;
public function cipher(): string;
```

### Exceptions

| Exception | Description |
|-----------|-------------|
| `EncryptionException` | Base exception for all encryption errors --- includes `getContext()` and `getSuggestion()` methods |
| `DecryptionException` | Thrown when decryption fails (invalid payload, wrong key, or tampered data) |

`DecryptionException` extends `EncryptionException` and provides static factory methods:

```php
use Marko\Encryption\Exceptions\DecryptionException;

DecryptionException::invalidPayload(); // corrupted or tampered data
DecryptionException::invalidKey();     // wrong encryption key
DecryptionException::invalidTagLength(actual: 1, expected: 16); // truncated authentication tag
DecryptionException::invalidIvLength(actual: 8, expected: 12);  // IV length does not match the cipher
```
