---
title: marko/broadcasting
description: Realtime broadcasting contracts --- send events to everyone watching a channel, with pluggable Mercure and Pusher drivers.
---

Realtime broadcasting contracts --- send an event to everyone watching a channel without coupling your app to how the connections are held. App code calls `BroadcasterInterface::broadcast()`; the installed driver delivers the event through a hub that holds the browser connections, so no PHP worker is tied up per viewer. This package defines the interfaces, the `Channel` / `PrivateChannel` value objects, and private-channel authorization via `#[BroadcastChannel]` authorizers. It ships no driver.

## Installation

```bash
composer require marko/broadcasting
```

You also need a driver:

| Driver | Holds connections in | Best for |
|---|---|---|
| [`marko/broadcasting-mercure`](/docs/packages/broadcasting-mercure/) | A Mercure hub (built into FrankenPHP/Caddy, or standalone) | Zero extra infrastructure on FrankenPHP; Server-Sent Events in the browser |
| [`marko/broadcasting-pusher`](/docs/packages/broadcasting-pusher/) | Hosted Pusher, Soketi, or Laravel Reverb | WebSockets, Laravel Echo / pusher-js clients |

Without a driver, resolving `BroadcasterInterface` throws `NoDriverException` listing the packages above.

:::tip
For a handful of viewers (admin dashboards, job progress), [`marko/sse`](/docs/packages/sse/) streams directly from PHP with no hub. See [When to Use SSE vs. Broadcasting](/docs/packages/sse/#when-to-use-sse-vs-broadcasting).
:::

## Usage

### Broadcasting an Event

Inject `BroadcasterInterface` and broadcast to a channel. A plain string is a public channel.

```php
use Marko\Broadcasting\BroadcasterInterface;
use Marko\Broadcasting\PrivateChannel;

class SeatService
{
    public function __construct(
        private BroadcasterInterface $broadcaster,
    ) {}

    public function sell(int $showId, string $seat): void
    {
        // ... persist the sale ...

        // Public: anyone watching the show sees the seat map update
        $this->broadcaster->broadcast("shows.$showId", 'seat.sold', ['seat' => $seat]);

        // Private: only subscribers authorized for this channel receive it
        $this->broadcaster->broadcast(
            new PrivateChannel("orders.$showId"),
            'order.updated',
            ['seat' => $seat],
            id: 'evt-1001',
        );
    }
}
```

`$data` must be JSON-serializable. The optional `$id` is passed to drivers that support event ids and replay (Mercure); the Pusher protocol has no event ids, so the Pusher driver does not transmit it.

### Broadcastable Event Objects

Implement `BroadcastableInterface` to describe an event once and broadcast it with `dispatch()`. It is sent to every listed channel. Nothing is broadcast implicitly --- implementing the interface does not hook into the event dispatcher.

```php
use Marko\Broadcasting\BroadcastableInterface;
use Marko\Broadcasting\PrivateChannel;

readonly class OrderShipped implements BroadcastableInterface
{
    public function __construct(
        public int $orderId,
        public int $customerId,
    ) {}

    public function channels(): array
    {
        return ['orders', new PrivateChannel("customers.$this->customerId")];
    }

    public function event(): string
    {
        return 'order.shipped';
    }

    public function payload(): array
    {
        return ['orderId' => $this->orderId];
    }
}

$broadcaster->dispatch(new OrderShipped(orderId: 7, customerId: 42));
```

### Authorizing Private Channels

Subscribers to a `PrivateChannel` must be authorized. Create a class implementing `ChannelAuthorizerInterface` and mark it with `#[BroadcastChannel]`. Classes anywhere in a module's `src/` directory are discovered automatically.

```php title="app/shop/src/Broadcasting/CustomerChannelAuthorizer.php"
use Marko\Authentication\AuthenticatableInterface;
use Marko\Broadcasting\Attributes\BroadcastChannel;
use Marko\Broadcasting\ChannelAuthorizerInterface;

#[BroadcastChannel('customers.{customerId}')]
class CustomerChannelAuthorizer implements ChannelAuthorizerInterface
{
    public function authorize(
        ?AuthenticatableInterface $user,
        array $params,
    ): bool {
        return $user !== null && (string) $user->getAuthIdentifier() === $params['customerId'];
    }
}
```

Patterns match dot-separated channel names. Each `{name}` placeholder matches exactly one segment and is passed to the authorizer as `$params['name']`, so `customers.{customerId}` matches `customers.42` but not `customers.4.2`. `$user` is `null` for guests. Authorizers are resolved from the container, so they can inject dependencies through their constructors.

Drivers call `ChannelRegistry::authorize($channelName, $user)` when they issue subscriber credentials (the Mercure subscriber token, the Pusher `/broadcasting/auth` endpoint). A private channel with no matching authorizer is **denied loudly** with a `ChannelAuthorizationException` --- private channels are never open by default.

### Testing

Use `FakeBroadcaster` from [`marko/testing`](/docs/packages/testing/#fakebroadcaster):

```php
use Marko\Testing\Fake\FakeBroadcaster;

$broadcaster = new FakeBroadcaster();
$service = new SeatService($broadcaster);

$service->sell(42, 'A1');

$broadcaster->assertBroadcast('shows.42', 'seat.sold', fn (array $data): bool => $data['seat'] === 'A1');
```

## API Reference

### BroadcasterInterface

```php
use Marko\Broadcasting\BroadcasterInterface;

/** @throws BroadcastException */
public function broadcast(string|Channel $channel, string $event, array $data, ?string $id = null): void;

/** @throws BroadcastException */
public function dispatch(BroadcastableInterface $broadcastable): void;
```

### Channel and PrivateChannel

```php
use Marko\Broadcasting\Channel;
use Marko\Broadcasting\PrivateChannel;

readonly class Channel
{
    public function __construct(public string $name); // throws BroadcastException when empty
    public static function from(string|Channel $channel): Channel;
    public function isPrivate(): bool; // false
}

readonly class PrivateChannel extends Channel {} // isPrivate(): true
```

### BroadcastableInterface

```php
use Marko\Broadcasting\BroadcastableInterface;

/** @return list<string|Channel> */
public function channels(): array;
public function event(): string;
/** @return array<string, mixed> */
public function payload(): array;
```

### #[BroadcastChannel] and ChannelAuthorizerInterface

```php
use Marko\Broadcasting\Attributes\BroadcastChannel;
use Marko\Broadcasting\ChannelAuthorizerInterface;

#[BroadcastChannel(pattern: 'shows.{showId}')]

/** @param array<string, string> $params */
public function authorize(?AuthenticatableInterface $user, array $params): bool;
```

### ChannelRegistry

```php
use Marko\Broadcasting\ChannelRegistry;

/** @throws ChannelAuthorizationException when no authorizer matches */
public function authorize(string $channelName, ?AuthenticatableInterface $user): bool;

/** @return list<string> */
public function patterns(): array;
```

A shared singleton; authorizers are discovered on first use.

### Exceptions

| Exception | Thrown when |
|---|---|
| `BroadcastException` | Empty channel or event name, unencodable payload, a driver request fails, or a channel name is invalid for the driver |
| `ChannelAuthorizationException` | A private channel has no authorizer, an authorizer is registered twice for one pattern, or a `#[BroadcastChannel]` class does not implement `ChannelAuthorizerInterface` |
| `NoDriverException` | `BroadcasterInterface` is resolved with no driver installed |

## Related Packages

- [`marko/broadcasting-mercure`](/docs/packages/broadcasting-mercure/) --- Mercure hub driver
- [`marko/broadcasting-pusher`](/docs/packages/broadcasting-pusher/) --- Pusher-protocol driver (Pusher, Soketi, Laravel Reverb)
- [`marko/sse`](/docs/packages/sse/) --- in-process Server-Sent Events for low-concurrency streams
- [`marko/testing`](/docs/packages/testing/) --- `FakeBroadcaster`
