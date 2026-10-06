---
title: marko/mail-log
description: Log-based mail driver — writes emails to the log instead of sending them, for development and testing.
---

Log-based mail driver --- writes emails to the log instead of sending them, for development and testing. Implements `MailerInterface` from [`marko/mail`](/docs/packages/mail/) by writing email details to the logger rather than delivering them. Envelope metadata (from, to, subject, attachment names) is logged at `info` level; in development, full HTML/text bodies are also logged at `debug` level. Every `send()` call returns `true` since no delivery can fail.

The driver refuses to boot in production: a log driver delivers no mail, and message bodies carry password-reset links and verification tokens that anyone with log access could use.

## Installation

Install it as a development dependency so it is never deployed:

```bash
composer require --dev marko/mail-log
```

This automatically installs [`marko/mail`](/docs/packages/mail/) and [`marko/log`](/docs/packages/log/).

## Usage

### Automatic via Binding

Installing the package binds `MailerInterface` to `LogMailer` --- no configuration is needed. Its `module.php` does the binding for you:

```php title="module.php"
use Marko\Mail\Contracts\MailerInterface;
use Marko\Mail\Log\LogMailer;

return [
    'bindings' => [
        MailerInterface::class => LogMailer::class,
    ],
];
```

Use a real driver such as [`marko/mail-smtp`](/docs/packages/mail-smtp/) in production. Because `marko/mail-log` is a `require-dev` dependency, `composer install --no-dev` leaves it out of production builds and the production driver's binding is the only one.

### Production Guard

If `marko/mail-log` is installed while the application runs in production (`MARKO_ENV`/`APP_ENV` set to `production` or `prod`, or unset), the application refuses to boot with a `MailLogException` explaining the risk. The guard checks whether the package is installed, not which mailer a binding resolves to: an app module that overrides `MailerInterface` for production still trips it, so keep the package out of production builds rather than overriding its binding.

To log mail in production deliberately, opt in:

```php title="config/mail-log.php"
return [
    'allow_production' => true,
];
```

or set `MAIL_LOG_ALLOW_PRODUCTION=true`. Bodies stay excluded in production unless you also set `include_body` to `true`.

### What Gets Logged

When you send a message in development, the log output includes:

```
[2025-01-15 10:30:00] app.INFO: Email sent {"from":"noreply@example.com","to":["user@example.com"],"subject":"Welcome!","has_html":true,"has_text":false,"attachment_count":0}
[2025-01-15 10:30:00] app.DEBUG: Email body (html) {"body":"<h1>Welcome!</h1>"}
```

The `info`-level entry captures envelope metadata --- from address, recipients, subject, whether HTML/text bodies are present, and attachment count. If the message includes CC, BCC, or attachments, those details are included as well (attachment name, size, and MIME type). When bodies are included, the `debug`-level entries log the full message body for each content type (text and/or HTML).

For `sendRaw()`, the `info` entry logs the recipient and raw content length; when bodies are included, a `debug` entry logs the full raw content.

Outside development, only the `info` entries are written.

Your code stays the same regardless of driver:

```php
use Marko\Mail\Contracts\MailerInterface;
use Marko\Mail\Message;

class WelcomeMailer
{
    public function __construct(
        private MailerInterface $mailer,
    ) {}

    public function sendWelcome(
        string $email,
    ): void {
        $message = Message::create()
            ->to($email)
            ->from('noreply@example.com')
            ->subject('Welcome!')
            ->html('<h1>Welcome!</h1>');

        $this->mailer->send($message);
    }
}
```

## Configuration

```php title="config/mail-log.php"
use Marko\Config\Env;

return [
    'include_body' => null,
    'allow_production' => Env::bool('MAIL_LOG_ALLOW_PRODUCTION', false),
];
```

| Key | Default | Description |
|---|---|---|
| `include_body` | `null` | Log text/HTML bodies and raw content at `debug` level. `null` logs them only in development (`development`, `dev`, `local`); `true` or `false` forces the choice in every environment. |
| `allow_production` | `false` | Let the application boot with this driver installed in production. |

## API Reference

### LogMailer

Implements `MailerInterface`. See [`marko/mail`](/docs/packages/mail/) for the full interface contract.

| Method | Description |
|---|---|
| `send(Message $message): bool` | Log envelope metadata at `info` level and, when bodies are included, each body at `debug` level; always returns `true` |
| `sendRaw(string $to, string $raw): bool` | Log recipient and raw content length at `info` level and, when bodies are included, the full content at `debug` level; always returns `true` |

The constructor takes the logger and the package config:

```php
use Marko\Log\Contracts\LoggerInterface;
use Marko\Mail\Log\Config\MailLogConfig;

public function __construct(
    private LoggerInterface $logger,
    private MailLogConfig $config,
) {}
```

### MailLogConfig

| Method | Description |
|---|---|
| `includeBody(): bool` | The resolved `mail-log.include_body`, falling back to "development only" when it is `null` |
| `allowProduction(): bool` | `mail-log.allow_production` |

### ProductionGuard

Runs from the module's `boot` callback. `check(): void` throws `MailLogException` when the environment is production and `allow_production` is `false`.
