---
title: marko/security
description: CSRF protection and security headers middleware -- secure your routes with drop-in middleware.
---

CSRF protection and security headers middleware --- secure your routes with drop-in middleware. Two middleware classes cover the most common web security needs: `CsrfMiddleware` validates tokens on state-changing requests, and `SecurityHeadersMiddleware` adds protective response headers (HSTS, CSP, X-Frame-Options, etc.). Both are configured via `config/security.php`.

For cross-origin requests, install [marko/cors](/docs/packages/cors/). `marko/security` used to ship its own `CorsMiddleware` with `security.cors.*` config; it was removed so there is a single CORS implementation. Replace any `#[Middleware]` reference to the old class with `marko/cors` (which registers itself globally) and move the `cors` block of `config/security.php` to `config/cors.php`.

## Installation

```bash
composer require marko/security
```

Requires [marko/session](/docs/packages/session/) and [marko/encryption](/docs/packages/encryption/) for CSRF token management.

## Configuration

Both middleware classes read from `config/security.php`:

```php title="config/security.php"
return [
    'csrf' => [
        'session_key' => '_csrf_token',
    ],
    'headers' => [
        'x_content_type_options' => 'nosniff',
        'x_frame_options' => 'SAMEORIGIN',
        'x_xss_protection' => '1; mode=block',
        'strict_transport_security' => 'max-age=31536000; includeSubDomains',
        'referrer_policy' => 'strict-origin-when-cross-origin',
        'content_security_policy' => "default-src 'self'",
    ],
];
```

## Usage

### CSRF Protection

Apply `CsrfMiddleware` to routes that accept form submissions:

```php
use Marko\Routing\Attributes\Post;
use Marko\Routing\Attributes\Middleware;
use Marko\Security\Middleware\CsrfMiddleware;

class FormController
{
    #[Post('/contact')]
    #[Middleware(CsrfMiddleware::class)]
    public function submit(): Response
    {
        // Token validated automatically
        return new Response('Submitted');
    }
}
```

The middleware checks `_token` in POST data or the `X-CSRF-TOKEN` header. Safe methods (GET, HEAD, OPTIONS) are skipped automatically.

You can also register `CsrfMiddleware` as global middleware in your module's `module.php`. It still never runs on requests that match no route, because it does not carry `#[RunsOnUnmatched]` (see [Which middleware runs](/docs/packages/routing/#which-middleware-runs)). A `POST` to an unknown path gets a `404`, and a `POST` to a GET-only path gets a `405` with `Allow` --- never a `419` token mismatch. The token check applies only to routes that exist.

Issuing a token (`CsrfTokenManagerInterface::get()` the first time) writes it to the session. That counts as a modification, so the session is saved and the visitor gets a session cookie even under [lazy session persistence](/docs/packages/session/#lazy-persistence). The form submission then carries the cookie that holds the token.

Include the token in forms:

```php
use Marko\Security\Contracts\CsrfTokenManagerInterface;

readonly class ContactController
{
    public function __construct(
        private CsrfTokenManagerInterface $csrfTokenManager,
    ) {}

    public function form(): Response
    {
        $token = $this->csrfTokenManager->get();
        // Render form with <input type="hidden" name="_token" value="$token">
    }
}
```

### Security Headers Middleware

Add protective HTTP headers to all responses:

```php
use Marko\Routing\Attributes\Middleware;
use Marko\Security\Middleware\SecurityHeadersMiddleware;

#[Middleware(SecurityHeadersMiddleware::class)]
```

Headers are configured in `config/security.php` under the `headers` key (see [Configuration](#configuration) above). Empty values are omitted from the response --- only headers with non-empty values are added.

### Using the CSRF Token Manager Directly

Regenerate tokens (e.g., after login):

```php
$newToken = $this->csrfTokenManager->regenerate();
```

Validate manually:

```php
if (!$this->csrfTokenManager->validate($submittedToken)) {
    // Invalid token
}
```

Tokens are generated using 32 random bytes encrypted via [marko/encryption](/docs/packages/encryption/) and stored in the session. The `validate()` method uses timing-safe comparison (`hash_equals`) to prevent timing attacks.

## Customization

Replace `CsrfTokenManager` via [Preferences](/docs/packages/core/) to change token generation or storage:

```php
use Marko\Core\Attributes\Preference;
use Marko\Security\CsrfTokenManager;

#[Preference(replaces: CsrfTokenManager::class)]
class MyCsrfTokenManager extends CsrfTokenManager
{
    public function get(): string
    {
        // Custom token retrieval logic
    }
}
```

## Exceptions

`CsrfMiddleware` throws `CsrfTokenMismatchException` when validation fails. This exception extends `SecurityException`, which provides rich context and suggestions --- consistent with Marko's loud-errors principle:

```php
use Marko\Security\Exceptions\CsrfTokenMismatchException;

// Thrown automatically by CsrfMiddleware:
// message:    "CSRF token validation failed."
// context:    "The submitted CSRF token does not match the token stored in the session..."
// suggestion: "Ensure your form includes a valid CSRF token field (_token) or X-CSRF-TOKEN header..."
```

`CsrfTokenMismatchException` implements `Marko\Core\Exceptions\HttpExceptionInterface`, so the routing pipeline renders it as **`419 Page Expired`** with the body `{"message": "CSRF token mismatch."}` (JSON, or a minimal HTML page for browsers). The detailed message, context and suggestion above stay server-side. Because the response is rendered where the middleware threw, outer middleware such as CORS and security headers still decorate it --- see [Errors and HTTP Exceptions](/docs/packages/routing/#errors-and-http-exceptions).

## API Reference

### CsrfTokenManagerInterface

```php
use Marko\Security\Contracts\CsrfTokenManagerInterface;

public function get(): string;       // Get token, generating one if none exists
public function validate(string $token): bool;  // Validate against stored token
public function regenerate(): string; // Regenerate token, replacing the previous one
```

### CsrfMiddleware

```php
use Marko\Security\Middleware\CsrfMiddleware;

public function handle(Request $request, callable $next): Response;
```

### SecurityHeadersMiddleware

```php
use Marko\Security\Middleware\SecurityHeadersMiddleware;

public function handle(Request $request, callable $next): Response;
```

### SecurityConfig

```php
use Marko\Security\Config\SecurityConfig;

public function csrfSessionKey(): string;
public function headerXContentTypeOptions(): string;
public function headerXFrameOptions(): string;
public function headerXXssProtection(): string;
public function headerStrictTransportSecurity(): string;
public function headerReferrerPolicy(): string;
public function headerContentSecurityPolicy(): string;
```

### SecurityException

```php
use Marko\Security\Exceptions\SecurityException;

public function getContext(): string;
public function getSuggestion(): string;
```

### CsrfTokenMismatchException

```php
use Marko\Security\Exceptions\CsrfTokenMismatchException;

public static function invalidToken(): self;
public function getStatusCode(): int;       // 419
public function getHeaders(): array;        // []
public function getResponseData(): array;   // ['message' => 'CSRF token mismatch.']
```
