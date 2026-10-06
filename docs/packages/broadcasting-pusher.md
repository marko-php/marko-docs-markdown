---
title: marko/broadcasting-pusher
description: Pusher-protocol broadcasting driver --- realtime events through hosted Pusher, Soketi, or Laravel Reverb.
---

Pusher-protocol broadcasting driver --- trigger realtime events through hosted Pusher or any server that speaks the Pusher protocol, such as [Soketi](https://soketi.app) or [Laravel Reverb](https://reverb.laravel.com). The server holds the WebSocket connections; PHP makes one signed HTTP request per event. Installing this package binds `BroadcasterInterface` from [`marko/broadcasting`](/docs/packages/broadcasting/) to `PusherBroadcaster` and registers the `POST /broadcasting/auth` endpoint that authorizes private channels. Requests are signed directly per the Pusher protocol, so `pusher/pusher-php-server` is not required.

## Installation

```bash
composer require marko/broadcasting-pusher
```

This installs `marko/broadcasting` and `marko/http`. You also need an HTTP client driver such as [`marko/http-guzzle`](/docs/packages/http-guzzle/).

## Configuration

```php title="config/broadcasting-pusher.php"
return [
    'app_id' => $_ENV['PUSHER_APP_ID'] ?? '',
    'key' => $_ENV['PUSHER_APP_KEY'] ?? '',
    'secret' => $_ENV['PUSHER_APP_SECRET'] ?? '',
    'cluster' => $_ENV['PUSHER_APP_CLUSTER'] ?? 'mt1',
    'host' => $_ENV['PUSHER_HOST'] ?? '',
    'port' => (int) ($_ENV['PUSHER_PORT'] ?? 443),
    'scheme' => $_ENV['PUSHER_SCHEME'] ?? 'https',
    'timeout' => 5,
];
```

When `host` is empty, the API host is derived from the cluster: `api-{cluster}.pusher.com`.

### Hosted Pusher

```bash title=".env"
PUSHER_APP_ID=123456
PUSHER_APP_KEY=your-key
PUSHER_APP_SECRET=your-secret
PUSHER_APP_CLUSTER=eu
```

### Soketi

```bash title=".env"
PUSHER_APP_ID=app-id
PUSHER_APP_KEY=app-key
PUSHER_APP_SECRET=app-secret
PUSHER_HOST=127.0.0.1
PUSHER_PORT=6001
PUSHER_SCHEME=http
```

### Laravel Reverb

Reverb speaks the same protocol. Point the driver at the Reverb server with the app credentials Reverb is configured with:

```bash title=".env"
PUSHER_APP_ID=my-app-id
PUSHER_APP_KEY=my-app-key
PUSHER_APP_SECRET=my-app-secret
PUSHER_HOST=127.0.0.1
PUSHER_PORT=8080
PUSHER_SCHEME=http
```

## Usage

### Publishing

Use `BroadcasterInterface` as described in [`marko/broadcasting`](/docs/packages/broadcasting/). Each broadcast is a signed `POST {scheme}://{host}:{port}/apps/{app_id}/events` with the body `{"name": event, "channels": [channel], "data": "<JSON data>"}`. A `PrivateChannel` is sent with the `private-` prefix (`orders.7` → `private-orders.7`).

Channel names may only contain letters, digits and `_ - = @ , . ;`, up to 164 characters including the `private-` prefix; other names throw `BroadcastException` before any request is made. The Pusher protocol has no event ids, so the `$id` argument is not transmitted.

### Client Setup

Use [pusher-js](https://github.com/pusher/pusher-js) directly or through Laravel Echo:

```javascript
import Echo from 'laravel-echo';
import Pusher from 'pusher-js';

window.Pusher = Pusher;

const echo = new Echo({
    broadcaster: 'pusher',
    key: 'your-key',
    cluster: 'eu',
    // For Soketi or Reverb, instead of cluster:
    // wsHost: '127.0.0.1', wsPort: 6001, forceTLS: false, enabledTransports: ['ws', 'wss'],
    authEndpoint: '/broadcasting/auth',
});

echo.channel('shows.42').listen('.seat.sold', (data) => markSeatSold(data.seat));
echo.private('orders.7').listen('.order.shipped', (data) => showShipped(data));
```

Echo prefixes event names with a namespace unless they start with `.`, so listen with a leading dot for the plain event names Marko sends.

### Private Channel Authorization

When a client subscribes to `private-*`, pusher-js `POST`s `socket_id` and `channel_name` to `/broadcasting/auth`. `PusherAuthController` strips the `private-` prefix, asks the [`ChannelRegistry`](/docs/packages/broadcasting/#authorizing-private-channels) whether the current user (from `GuardInterface`) may subscribe, and responds:

| Situation | Response |
|---|---|
| Authorized | `200 {"auth": "key:HMAC-SHA256(secret, socket_id:channel_name)"}` |
| The authorizer denies the user | `403` |
| Missing/malformed `socket_id`, or a channel without the `private-` prefix | `400` |
| No authorizer matches the channel | `ChannelAuthorizationException` (loud error) |
| A `presence-*` channel | `PusherException` --- presence channels are not supported yet |

Package routes are always discovered. To change the path or add middleware, replace the controller with a [`#[Preference]`](/docs/concepts/preferences/) subclass:

```php title="app/shop/src/Controller/BroadcastAuthController.php"
use Marko\Broadcasting\Pusher\Controller\PusherAuthController;
use Marko\Core\Attributes\Preference;
use Marko\Routing\Attributes\Middleware;
use Marko\Routing\Attributes\Post;
use Marko\Routing\Http\Request;
use Marko\Routing\Http\Response;
use Marko\Authentication\Middleware\AuthMiddleware;

#[Preference(replaces: PusherAuthController::class)]
readonly class BroadcastAuthController extends PusherAuthController
{
    #[Post('/api/broadcasting/auth')]
    #[Middleware(AuthMiddleware::class)]
    public function authorize(
        Request $request,
    ): Response {
        return parent::authorize($request);
    }
}
```

## API Reference

### PusherBroadcaster

Implements `BroadcasterInterface`.

```php
use Marko\Broadcasting\Pusher\Driver\PusherBroadcaster;

public function broadcast(string|Channel $channel, string $event, array $data, ?string $id = null): void;
public function dispatch(BroadcastableInterface $broadcastable): void;
```

Each request is signed with `auth_timestamp` read from the injected PSR-20 [`ClockInterface`](/docs/packages/clock/). In tests, a [`FakeClock`](/docs/packages/testing/#fakeclock) makes the signed query string predictable.

### PusherSignature

```php
use Marko\Broadcasting\Pusher\Auth\PusherSignature;

/** @param array<string, string> $query */
public function sign(string $method, string $path, array $query): string;

/** @return array<string, string> auth_key, auth_timestamp, auth_version, body_md5, auth_signature */
public function signedQuery(string $method, string $path, string $body, int $timestamp): array;

public function channelAuth(string $socketId, string $channelName): string;
```

### PusherAuthController

```php
use Marko\Broadcasting\Pusher\Controller\PusherAuthController;

#[Post('/broadcasting/auth')]
public function authorize(Request $request): Response;
```

### PusherConfig

Readonly value object built from `config/broadcasting-pusher.php` by the module binding; `apiHost()` and `baseUrl()` derive the API endpoint.

### Exceptions

| Exception | Thrown when |
|---|---|
| `PusherException::missingCredentials()` | `app_id`, `key` or `secret` is empty when signing |
| `PusherException::presenceChannelsNotSupported()` | A client requests authorization for a `presence-*` channel |
| `BroadcastException::invalidChannelName()` | A channel name contains characters the Pusher protocol does not allow |
| `BroadcastException::publishFailed()` | The server is unreachable or answers with a non-2xx status |

## Related Packages

- [`marko/broadcasting`](/docs/packages/broadcasting/) --- the interfaces and channel authorization
- [`marko/broadcasting-amphp`](/docs/packages/broadcasting-amphp/) --- self-hosted async SSE server driver
- [`marko/broadcasting-mercure`](/docs/packages/broadcasting-mercure/) --- Mercure hub driver
- [`marko/http-guzzle`](/docs/packages/http-guzzle/) --- HTTP client driver used to reach the API
