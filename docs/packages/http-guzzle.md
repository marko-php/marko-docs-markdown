---
title: marko/http-guzzle
description: Guzzle-powered HTTP client driver — makes real HTTP requests using the battle-tested Guzzle library.
---

Guzzle-powered HTTP client driver --- makes real HTTP requests using the battle-tested Guzzle library. Implements `HttpClientInterface` from [`marko/http`](/docs/packages/http/) using Guzzle under the hood. Connection failures throw `ConnectionException`; HTTP errors throw `HttpException` with the response attached.

## Installation

```bash
composer require marko/http-guzzle
```

This automatically installs `marko/http`.

## Usage

### Automatic via Binding

Bind the interface to the Guzzle implementation in your `module.php`:

```php title="module.php"
use Marko\Http\Contracts\HttpClientInterface;
use Marko\Http\Guzzle\GuzzleHttpClient;

return [
    'bindings' => [
        HttpClientInterface::class => GuzzleHttpClient::class,
    ],
];
```

Then inject `HttpClientInterface` anywhere:

```php
use Marko\Http\Contracts\HttpClientInterface;

class ApiClient
{
    public function __construct(
        private HttpClientInterface $httpClient,
    ) {}

    public function fetchData(): array
    {
        $response = $this->httpClient->get('https://api.example.com/data', [
            'timeout' => 10,
            'headers' => ['Accept' => 'application/json'],
        ]);

        return $response->json();
    }
}
```

### Handling Errors

```php
use Marko\Http\Exceptions\ConnectionException;
use Marko\Http\Exceptions\HttpException;

try {
    $response = $this->httpClient->get('https://api.example.com/resource');
} catch (ConnectionException $e) {
    // Network failure (DNS, timeout, etc.)
} catch (HttpException $e) {
    // HTTP error (4xx, 5xx) --- response may be available
    $errorResponse = $e->getResponse();
}
```

Pass `'http_errors' => false` to receive 4xx/5xx responses as a normal `HttpResponse` instead of an exception. See [Error Responses](/docs/packages/http/#error-responses).

### Request Options and Validation

`GuzzleHttpClient` accepts the portable option set documented in [`marko/http`](/docs/packages/http/#request-options) and validates it with `RequestOptions::validate()` before sending. An unknown key, conflicting body options, or a malformed value throws `InvalidRequestOptionException` --- nothing is silently dropped.

Portable options map to Guzzle as follows:

| Option | Sent to Guzzle as |
|---|---|
| `headers`, `body`, `json`, `form_params`, `multipart`, `query`, `timeout`, `connect_timeout`, `verify`, `proxy` | The Guzzle option of the same name |
| `auth` `['user', 'pass']` | Guzzle `auth` (HTTP basic) |
| `auth` `['bearer' => $token]` | An `Authorization: Bearer $token` header |
| `allow_redirects` `true`/`false` | Guzzle `allow_redirects` |
| `allow_redirects` `int` | Guzzle `allow_redirects` with `['max' => $int]` |
| `http_errors` | Guzzle `http_errors` (default `true`) |

### The `guzzle` Escape Hatch

For a Guzzle feature outside the portable set (`cert`, `ssl_key`, `sink`, `on_stats`, `decode_content`, `debug`, ...), pass a `guzzle` array. It is merged verbatim over the options above, so it wins on conflict:

```php
$response = $this->httpClient->get('https://internal.example.com/report.csv', [
    'timeout' => 30,
    'guzzle' => [
        'cert' => '/etc/ssl/client.pem',
        'sink' => '/tmp/report.csv',
    ],
]);
```

The `guzzle` key is **not portable**: any other `HttpClientInterface` driver, and `FakeHttpClient`, reject it with `InvalidRequestOptionException`. Keeping it at the call site makes the reach past the interface explicit. The constant `GuzzleHttpClient::GUZZLE_OPTIONS` holds the key name.

### Response Headers

Guzzle exposes each response header as a list of values. `HttpResponse::headers()` returns one string per header, so repeated values are joined with `", "`. This is lossy for `Set-Cookie`, whose values can themselves contain commas (for example in `Expires`). If you need each cookie separately, read the raw response through a custom `createClient()` middleware.

## Customization

Extend `GuzzleHttpClient` via Preference to customize the underlying Guzzle client:

```php
use Marko\Core\Attributes\Preference;
use Marko\Http\Guzzle\GuzzleHttpClient;
use GuzzleHttp\Client;
use GuzzleHttp\ClientInterface as GuzzleClientInterface;

#[Preference(replaces: GuzzleHttpClient::class)]
class CustomGuzzleClient extends GuzzleHttpClient
{
    protected function createClient(): GuzzleClientInterface
    {
        return new Client([
            'base_uri' => 'https://api.example.com',
            'timeout' => 30,
        ]);
    }
}
```

The `createClient()` method is called lazily on first request --- override it to set base URIs, default timeouts, middleware, or any other Guzzle configuration.

## API Reference

### GuzzleHttpClient

Implements `HttpClientInterface`. See [`marko/http`](/docs/packages/http/) for the full contract.

| Method | Description |
|---|---|
| `request(string $method, string $url, array $options = []): HttpResponse` | Send a request with any HTTP method |
| `get(string $url, array $options = []): HttpResponse` | Send a GET request |
| `post(string $url, array $options = []): HttpResponse` | Send a POST request |
| `put(string $url, array $options = []): HttpResponse` | Send a PUT request |
| `patch(string $url, array $options = []): HttpResponse` | Send a PATCH request |
| `delete(string $url, array $options = []): HttpResponse` | Send a DELETE request |
| `createClient(): GuzzleClientInterface` | Protected --- override via Preference to customize the Guzzle client |

All methods throw `InvalidRequestOptionException` for invalid options, `ConnectionException` on network failures, and `HttpException` on HTTP error responses (4xx, 5xx) unless `http_errors` is `false`. The `HttpException` carries the `HttpResponse` when available.
