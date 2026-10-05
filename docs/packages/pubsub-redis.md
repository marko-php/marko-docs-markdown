---
title: marko/pubsub-redis
description: Non-blocking Redis pub/sub driver — publish and subscribe over Redis with pattern support, powered by amphp for async I/O.
---

Non-blocking Redis pub/sub for Marko --- publish and subscribe over Redis with pattern support, powered by amphp for async I/O. Provides `RedisPublisher` and `RedisSubscriber`, implementing the `PublisherInterface` and `SubscriberInterface` contracts from [`marko/pubsub`](/docs/packages/pubsub/). Uses `amphp/redis` for non-blocking Redis connections so the subscriber loop never stalls. Pattern subscriptions (glob-style channel matching) are fully supported.

Installing this package binds `PublisherInterface` and `SubscriberInterface` to the Redis driver automatically --- no manual wiring required.

## Installation

```bash
composer require marko/pubsub-redis
```

This automatically installs `marko/pubsub` and `marko/amphp`.

## Configuration

The package ships `config/pubsub-redis.php`, and its module binding builds a single shared `RedisPubSubConnection` from it. Set the environment variables, or override the file in your app:

```php title="config/pubsub-redis.php"
return [
    'host' => $_ENV['PUBSUB_REDIS_HOST'] ?? '127.0.0.1',
    'port' => (int) ($_ENV['PUBSUB_REDIS_PORT'] ?? 6379),
    'password' => $_ENV['PUBSUB_REDIS_PASSWORD'] ?? null,
    'database' => (int) ($_ENV['PUBSUB_REDIS_DATABASE'] ?? 0),
];
```

| Key | Env var | Default | Description |
|---|---|---|---|
| `host` | `PUBSUB_REDIS_HOST` | `127.0.0.1` | Redis server host |
| `port` | `PUBSUB_REDIS_PORT` | `6379` | Redis server port |
| `password` | `PUBSUB_REDIS_PASSWORD` | `null` | Password for `AUTH`; `null` or empty means no authentication |
| `database` | `PUBSUB_REDIS_DATABASE` | `0` | Redis database index |

The channel prefix is not a Redis setting. It comes from `pubsub.prefix` in [`marko/pubsub`](/docs/packages/pubsub/) (`PUBSUB_PREFIX`, default `marko:`), which is the single source of truth: the publisher, the subscriber and `RedisPubSubConnection::$prefix` all use it.

```bash
PUBSUB_DRIVER=redis
PUBSUB_PREFIX=marko:
```

## Usage

### Publishing

Inject `PublisherInterface` --- the Redis driver is used automatically:

```php
use Marko\PubSub\Message;
use Marko\PubSub\PublisherInterface;

class NotificationService
{
    public function __construct(
        private PublisherInterface $publisher,
    ) {}

    public function notify(int $userId, string $text): void
    {
        $this->publisher->publish(
            channel: "user.$userId",
            message: new Message(
                channel: "user.$userId",
                payload: json_encode(['text' => $text]),
            ),
        );
    }
}
```

### Subscribing

Inject `SubscriberInterface` and iterate the `Subscription`. Pass multiple channel names to receive from all of them in a single subscription. Run the subscriber loop via the `pubsub:listen` command:

```php
use Marko\PubSub\SubscriberInterface;

class NotificationListener
{
    public function __construct(
        private SubscriberInterface $subscriber,
    ) {}

    public function listen(int $userId): void
    {
        // Single channel
        $subscription = $this->subscriber->subscribe("user.$userId");

        foreach ($subscription as $message) {
            $data = json_decode($message->payload, true);
            // handle notification ...
        }
    }

    public function listenAll(): void
    {
        // Multiple channels — delivers from all of them
        $subscription = $this->subscriber->subscribe('orders', 'shipments', 'returns');

        foreach ($subscription as $message) {
            // $message->channel tells you which channel delivered the message
            $data = json_decode($message->payload, true);
        }
    }
}
```

Start the listener process:

```bash
marko pubsub:listen
```

### Pattern Subscriptions

Use `psubscribe()` to receive messages from all channels matching a glob pattern:

```php
$subscription = $this->subscriber->psubscribe('user.*');

foreach ($subscription as $message) {
    // $message->pattern === 'user.*'
    // $message->channel is the matched channel, e.g. 'user.42'
    $data = json_decode($message->payload, true);
}
```

### SSE Integration

Combine with `marko/sse` to stream pub/sub messages to the browser:

```php
use Marko\PubSub\SubscriberInterface;
use Marko\Routing\Attributes\Get;
use Marko\Sse\SseStream;
use Marko\Sse\StreamingResponse;

#[Get('/users/{userId}/notifications')]
public function stream(int $userId): StreamingResponse
{
    $subscription = $this->subscriber->subscribe("user.$userId");

    $stream = new SseStream(
        subscription: $subscription,
        timeout: 300,
    );

    return new StreamingResponse($stream);
}
```

## Customization

To change how the connection is built (for example, to connect over a Unix socket), extend `RedisPubSubConnection` and override `redisConfig()`, which both the client and the connector use:

```php
use Amp\Redis\RedisConfig;
use Marko\PubSub\Redis\RedisPubSubConnection;

class SocketRedisPubSubConnection extends RedisPubSubConnection
{
    protected function redisConfig(): RedisConfig
    {
        return RedisConfig::fromUri("unix://$this->host")
            ->withDatabase($this->database)
            ->withPassword($this->password ?? '');
    }
}
```

Bind it in your app module with the same config keys the package binding reads:

```php title="app/mymodule/module.php"
use Marko\Config\ConfigRepositoryInterface;
use Marko\Core\Container\ContainerInterface;
use Marko\PubSub\Redis\RedisPubSubConnection;

return [
    'bindings' => [
        RedisPubSubConnection::class => static function (ContainerInterface $container): RedisPubSubConnection {
            $config = $container->get(ConfigRepositoryInterface::class);

            return new SocketRedisPubSubConnection(
                host: $config->getString(key: 'pubsub-redis.host'),
                port: $config->getInt(key: 'pubsub-redis.port'),
                password: $config->get(key: 'pubsub-redis.password'),
                database: $config->getInt(key: 'pubsub-redis.database'),
                prefix: $config->getString(key: 'pubsub.prefix'),
            );
        },
    ],
];
```

## API Reference

### RedisPublisher

| Method | Description |
|---|---|
| `__construct(RedisPubSubConnection $redisPubSubConnection, PubSubConfig $pubSubConfig)` | Create a publisher with a Redis connection and pub/sub configuration |
| `publish(string $channel, Message $message): void` | Publish a message to the given channel |

### RedisSubscriber

| Method | Description |
|---|---|
| `__construct(RedisPubSubConnection $redisPubSubConnection, PubSubConfig $pubSubConfig)` | Create a subscriber with a Redis connection and pub/sub configuration |
| `subscribe(string ...$channels): Subscription` | Subscribe to one or more channels, returning an iterable `Subscription` |
| `psubscribe(string ...$patterns): Subscription` | Subscribe to channels matching glob patterns, returning an iterable `Subscription` |

### RedisSubscription

| Method | Description |
|---|---|
| `__construct(AmphpRedisSubscription[] $amphpSubscriptions, string $prefix, string[] $channels, string[] $patterns)` | Wrap one or more amphp subscriptions with prefix stripping and message conversion. Pass channel names for channel subscriptions, pattern names for pattern subscriptions. |
| `getIterator(): Generator` | Yield `Message` instances from all subscriptions --- includes `pattern` and resolved `channel` for pattern subscriptions |
| `cancel(): void` | Unsubscribe from all channels/patterns and stop iteration |

### RedisPubSubConnection

| Method | Description |
|---|---|
| `__construct(string $host, int $port, ?string $password, int $database, string $prefix)` | Create a connection with host (`127.0.0.1`), port (`6379`), optional password, database index (`0`), and channel prefix (`marko:`). The module binding fills these from `config/pubsub-redis.php` and `pubsub.prefix` |
| `client(): RedisClient` | Get the Redis client instance --- lazily connected on first call |
| `connector(): RedisConnector` | Get the Redis connector instance --- lazily created on first call |
| `redisConfig(): RedisConfig` (protected) | Build the amphp `RedisConfig` (host, port, password, database) used by the client and connector |
| `disconnect(): void` | Disconnect and release both client and connector instances |
| `isConnected(): bool` | Check whether a client instance is currently active |
