---
title: marko/webhook
description: Send and receive webhooks with HMAC-SHA256 signature verification, automatic retry with exponential backoff, and delivery attempt tracking.
---

Send and receive webhooks with HMAC-SHA256 signature verification, automatic retry with exponential backoff, and delivery attempt tracking. Outgoing webhooks are signed with a shared secret and delivered over HTTP. Incoming webhooks are verified against the same signature before the payload is parsed. Failed deliveries are automatically retried via the queue with exponential backoff. Every delivery attempt --- success or failure --- is recorded to the `webhook_attempts` table.

## Installation

```bash
composer require marko/webhook
```

Requires [marko/http](/docs/packages/http/) for the HTTP client and [marko/queue](/docs/packages/queue/) for async dispatch.

## Configuration

Override defaults in your config file:

```php title="config/webhook.php"
return [
    'timeout'             => 30,  // seconds before an outgoing webhook request is abandoned; must be > 0
    'max_retries'         => 3,   // maximum delivery attempts (including the first)
    'retry_delay'         => 60,  // base delay in seconds; multiplied exponentially per attempt
    'timestamp_tolerance' => 300, // seconds a webhook timestamp may differ from server time
];
```

With the defaults, a job that fails on every attempt retries at 120 s, 240 s, and 480 s.

`WebhookDispatcher` passes `timeout` to the HTTP client on every delivery, so a receiver that accepts the connection and never answers fails the attempt after that many seconds instead of blocking the queue worker. The timed-out attempt is recorded as a failure and retried like any other transport error. `timeout` must be a positive integer: `0` would mean "wait forever" to the HTTP client, so `WebhookConfig` throws a `ConfigException` naming `webhook.timeout` for `0` or a negative value. That check runs wherever `WebhookConfig` is resolved, so a bad value fails loudly both when sending and when receiving webhooks. A missing `timeout` throws `ConfigNotFoundException`.

The `timestamp_tolerance` controls replay-attack protection for inbound webhooks. Requests whose `X-Webhook-Timestamp` header is more than this many seconds in the past or future are rejected.

## Usage

### Sending Webhooks

Build a `WebhookPayload` and call `WebhookDispatcher::dispatch()` to send synchronously:

```php
use Marko\Webhook\Sending\WebhookDispatcher;
use Marko\Webhook\Value\WebhookPayload;

public function __construct(
    private readonly WebhookDispatcher $webhookDispatcher,
) {}

public function notifySubscriber(): void
{
    $payload = new WebhookPayload(
        url: 'https://example.com/webhooks',
        event: 'order.created',
        data: ['order_id' => 42, 'total' => '99.99'],
        secret: 'your-shared-secret',
    );

    $response = $this->webhookDispatcher->dispatch($payload);

    if (!$response->successful) {
        // The receiver answered with a non-2xx status:
        // $response->statusCode and $response->body say why.
    }
}
```

`dispatch()` returns a `WebhookResponse` for every HTTP response, 4xx and 5xx included, so check `$response->successful` (true only for 2xx). It throws only when the receiver cannot be reached at all (`ConnectionException`/`HttpException` from [`marko/http`](/docs/packages/http/)). `$response->isRetryable()` tells you whether sending again could help: true for `408`, `429` and any `5xx`.

The dispatcher automatically records the current Unix timestamp, includes it as an `X-Webhook-Timestamp` header, and signs the message as `"{timestamp}.{body}"` — producing an `X-Webhook-Signature: sha256={hash}` header. The receiver verifies both the timestamp freshness and the signature before accepting the payload.

### Sending Asynchronously with Retry

Push a `DispatchWebhookJob` onto the queue to send in the background with automatic retry on failure:

```php
use Marko\Queue\QueueInterface;
use Marko\Webhook\Jobs\DispatchWebhookJob;
use Marko\Webhook\Value\WebhookPayload;

public function __construct(
    private readonly QueueInterface $queue,
) {}

public function scheduleWebhook(): void
{
    $payload = new WebhookPayload(
        url: 'https://example.com/webhooks',
        event: 'order.shipped',
        data: ['order_id' => 42, 'tracking' => 'ABC123'],
        secret: 'your-shared-secret',
    );

    $this->queue->push(new DispatchWebhookJob($payload));
}
```

The job decides what to do from the outcome of each attempt:

| Outcome | Recorded as | Retried |
|---|---|---|
| `2xx` response | Success: status code and response body (capped at 500 bytes) | No |
| `408`, `429` or `5xx` response | Failure: status code, response body (capped at 500 bytes) and `Webhook receiver responded with HTTP {status}.` | Yes |
| Any other non-2xx response (`3xx`, other `4xx`) | Failure, same fields | No. The receiver will answer the same way again |
| Receiver unreachable or too slow (connection error, transport error, or no answer within `timeout` seconds) | Failure: the error message | Yes |

Retries go back onto the queue with `retry_delay * 2^attempt` seconds of delay until `max_retries` attempts have been made. When `max_retries` or `retry_delay` is missing from config, the attempt is still recorded but not retried. Only the send is retried: if recording a delivered webhook fails (for example, the database is down), the exception propagates instead of sending the webhook again.

### Receiving Webhooks

Use `WebhookReceiver::receive()` in a controller to verify the signature and parse the payload. An `InvalidSignatureException` is thrown if the `X-Webhook-Timestamp` header is absent, if the timestamp is outside the tolerance window, or if the signature does not match:

```php
use Marko\Routing\Http\Request;
use Marko\Webhook\Exceptions\InvalidSignatureException;
use Marko\Webhook\Receiving\WebhookReceiver;

public function __construct(
    private readonly WebhookReceiver $webhookReceiver,
) {}

public function handle(
    Request $request,
): void {
    try {
        $data = $this->webhookReceiver->receive(
            request: $request,
            secret: 'your-shared-secret',
        );

        $event = $data['event'];
        $payload = $data['data'];

        // process event...
    } catch (InvalidSignatureException) {
        // reject the request
    }
}
```

### Using the WebhookEndpoint Attribute

Mark a controller method with `#[WebhookEndpoint]` to declare its path and secret inline:

```php
use Marko\Routing\Http\Request;
use Marko\Webhook\Attributes\WebhookEndpoint;

class StripeWebhookController
{
    #[WebhookEndpoint(path: '/webhooks/stripe', secret: 'whsec_...')]
    public function handle(
        Request $request,
    ): void {
        // $request is already routed here; verify with WebhookReceiver
    }
}
```

### Delivery Tracking

Every attempt is saved to the `webhook_attempts` table via `WebhookDeliveryService`. Successful attempts store the HTTP status code and response body. Attempts the receiver rejected with a non-2xx status store the status code, the response body and an error message. Attempts that never reached the receiver, or got no answer within `timeout` seconds, store only the error message.

Every recorded response body, successful or rejected, is trimmed and capped at 500 bytes. A longer body is cut at a UTF-8 character boundary and ends with `... [truncated N bytes]`, so a receiver that returns a large page cannot overflow the `response_body` column after the webhook was already delivered. Use `WebhookAttemptRepositoryInterface` to query the records:

```php
use Marko\Webhook\Contracts\WebhookAttemptRepositoryInterface;

public function __construct(
    private readonly WebhookAttemptRepositoryInterface $webhookAttemptRepository,
) {}
```

## API Reference

### WebhookPayload

```php
use Marko\Webhook\Value\WebhookPayload;

public function __construct(
    string $url,
    string $event,
    array $data,
    string $secret,
);
```

### WebhookResponse

```php
use Marko\Webhook\Value\WebhookResponse;

public function __construct(
    int $statusCode,
    string $body,
    bool $successful,   // true only for 2xx
);
public function isRetryable(): bool;  // true for 408, 429 and 5xx
```

### WebhookDispatcher

```php
use Marko\Http\Contracts\HttpClientInterface;
use Marko\Webhook\Config\WebhookConfig;
use Marko\Webhook\Sending\WebhookDispatcher;
use Marko\Webhook\Value\WebhookPayload;
use Marko\Webhook\Value\WebhookResponse;
use Psr\Clock\ClockInterface;

public function __construct(
    HttpClientInterface $httpClient,
    ClockInterface $clock,
    WebhookConfig $config,
);

// Returns a WebhookResponse for every HTTP response (4xx/5xx included).
// Sends webhook.timeout as the request timeout.
// @throws HttpException|ConnectionException when the receiver cannot be reached or does not answer in time
public function dispatch(WebhookPayload $payload): WebhookResponse;
```

### WebhookReceiver

```php
use Marko\Routing\Http\Request;
use Marko\Webhook\Receiving\WebhookReceiver;

// @throws InvalidSignatureException
public function receive(Request $request, string $secret): array;
```

### WebhookVerifier

```php
use Marko\Webhook\Receiving\WebhookVerifier;

public function verify(string $body, string $timestamp, string $signature, string $secret, int $tolerance): bool;
```

### WebhookSignature

```php
use Marko\Webhook\Sending\WebhookSignature;

// Returns "sha256={hash}"; message is "{timestamp}.{payload}"
public static function sign(string $payload, string $secret, int $timestamp): string;
```

### DispatchWebhookJob

```php
use Marko\Webhook\Jobs\DispatchWebhookJob;
use Marko\Webhook\Value\WebhookPayload;

public function __construct(
    WebhookPayload $payload,
    int $attemptNumber = 1,
);
public function handle(): void;
```

Implements `ContainerAwareJobInterface` from [`marko/queue`](/docs/packages/queue/). The job stores only the `WebhookPayload` value object and the attempt number; services (`WebhookDispatcherInterface`, `WebhookDeliveryService`, `QueueInterface`, config) are resolved from the container at `handle()` time by the queue `Worker`, which releases the container (`releaseContainer()`) once `handle()` returns or throws, so a job that fails for the last time can still be serialized into the failed-job store. Do not inject services via the constructor.

### WebhookDeliveryService

```php
use Marko\Webhook\Sending\WebhookDeliveryService;
use Marko\Webhook\Value\WebhookPayload;
use Marko\Webhook\Value\WebhookResponse;

// Both store the response body trimmed and capped at 500 bytes (HttpResponse::bodyExcerpt())
public function recordSuccess(WebhookPayload $payload, WebhookResponse $response, int $attempt): void;   // 2xx
public function recordRejection(WebhookPayload $payload, WebhookResponse $response, int $attempt): void; // non-2xx
public function recordFailure(WebhookPayload $payload, string $error, int $attempt): void;
```

### WebhookAttempt (Entity)

| Column           | Type   | Description                        |
|------------------|--------|------------------------------------|
| `id`             | int    | Auto-increment primary key         |
| `webhook_url`    | string | Destination URL                    |
| `event`          | string | Event name                         |
| `attempt_number` | int    | Which attempt this record covers   |
| `status_code`    | int    | HTTP status code (null when the receiver was unreachable) |
| `response_body`  | string | Response body, trimmed and capped at 500 bytes (null when unreachable or timed out) |
| `error_message`  | string | Error message (failures and rejections only) |
| `attempted_at`   | string | Timestamp in `Y-m-d H:i:s` format, in the [database timezone](/docs/packages/database/#datetimes-and-timezones) (UTC by default) |

### WebhookConfig

```php
use Marko\Webhook\Config\WebhookConfig;

public int $timeout;             // from webhook.timeout; ConfigException when not positive
public int $maxRetries;          // from webhook.max_retries
public int $retryDelay;          // from webhook.retry_delay
public int $timestampTolerance;  // from webhook.timestamp_tolerance
```

### InvalidSignatureException

```php
use Marko\Webhook\Exceptions\InvalidSignatureException;

// Thrown when X-Webhook-Timestamp header is absent
InvalidSignatureException::missingTimestamp();

// Thrown when the timestamp is outside the tolerance window
InvalidSignatureException::staleTimestamp(int $timestamp, int $tolerance);

// Thrown when the HMAC signature does not match
InvalidSignatureException::forRequest();
```

### Interfaces

```php
use Marko\Webhook\Contracts\WebhookDispatcherInterface;
use Marko\Webhook\Value\WebhookPayload;
use Marko\Webhook\Value\WebhookResponse;

interface WebhookDispatcherInterface {
    public function dispatch(WebhookPayload $payload): WebhookResponse;
}
```

```php
use Marko\Routing\Http\Request;
use Marko\Webhook\Contracts\WebhookReceiverInterface;

interface WebhookReceiverInterface {
    public function receive(Request $request, string $secret): array;
}
```

```php
use Marko\Webhook\Contracts\WebhookAttemptRepositoryInterface;
use Marko\Webhook\Entity\WebhookAttempt;

interface WebhookAttemptRepositoryInterface {
    public function save(WebhookAttempt $attempt): WebhookAttempt;
}
```
