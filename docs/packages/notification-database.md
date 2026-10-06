---
title: marko/notification-database
description: Database notification storage — persist, query, and manage notification read state in the database.
---

Database notification storage --- persist, query, and manage notification read state in the database. Provides the `DatabaseNotification` entity, `NotificationRepositoryInterface`, and a `DatabaseNotificationRepository` implementation. Query a user's notifications, mark them as read, count unread notifications, and clean up old ones.

Works with the database channel from [`marko/notification`](/docs/packages/notification/) and requires [`marko/database`](/docs/packages/database/) for the database connection.

## Installation

```bash
composer require marko/notification-database
```

## Usage

### Querying Notifications

Inject the repository to fetch notifications for a notifiable entity:

```php
use Marko\Notification\Database\Repository\NotificationRepositoryInterface;

class NotificationController
{
    public function __construct(
        private NotificationRepositoryInterface $notificationRepository,
    ) {}

    public function index(
        User $user,
    ): array {
        return $this->notificationRepository->forNotifiable($user);
    }

    public function unreadCount(
        User $user,
    ): int {
        return $this->notificationRepository->unreadCount($user);
    }
}
```

### Marking as Read

Mark individual notifications or all at once:

```php
// Mark one of the user's notifications as read
if (!$this->notificationRepository->markAsReadFor($user, $notificationId)) {
    // Not found, or it belongs to someone else: respond 404
}

// Mark all notifications as read for a user
$this->notificationRepository->markAllAsRead($user);
```

`markAsReadFor()` only touches a notification whose `notifiable_type` and `notifiable_id` match the given notifiable, and returns `false` when there is no such notification. Use it whenever the ID comes from a request (`POST /notifications/{id}/read`): otherwise anyone who learns another user's notification ID could mark it as read. A notification that is already read keeps its original `read_at` and still returns `true`.

`markAsRead(string $notificationId)` marks a notification by ID alone, whoever owns it. Keep it for admin and internal code that has already decided the caller may act on that notification.

`read_at` is read from the injected PSR-20 [`ClockInterface`](/docs/packages/clock/) and written in the [database timezone](/docs/packages/database/#datetimes-and-timezones) (`database.timezone`, UTC by default), the same zone `DatabaseChannel` writes `created_at` in. In tests, construct `DatabaseNotificationRepository` with a [`FakeClock`](/docs/packages/testing/#fakeclock) and `DatabaseTimezoneConfig::fromName('UTC')` to get a known timestamp.

### Fetching Unread Notifications

```php
$unread = $this->notificationRepository->unread($user);

foreach ($unread as $notification) {
    $data = json_decode($notification->data, true);
    // Process notification data
}
```

### Deleting Notifications

```php
// Delete one of the user's notifications
if (!$this->notificationRepository->deleteFor($user, $notificationId)) {
    // Not found, or it belongs to someone else: respond 404
}

// Delete all notifications for a user
$this->notificationRepository->deleteAll($user);
```

As with marking as read, `deleteFor()` is the default for request-driven code: it deletes only a notification the notifiable owns and returns `false` otherwise. `delete(string $notificationId)` deletes by ID alone and is for admin and internal use.

## Customization

Replace the repository via Preference to add custom query logic:

```php
use Marko\Core\Attributes\Preference;
use Marko\Notification\Database\Repository\DatabaseNotificationRepository;
use Marko\Notification\Contracts\NotifiableInterface;
use Marko\Notification\Database\Entity\DatabaseNotification;

#[Preference(replaces: DatabaseNotificationRepository::class)]
class CustomNotificationRepository extends DatabaseNotificationRepository
{
    /**
     * @return array<DatabaseNotification>
     */
    public function forNotifiable(
        NotifiableInterface $notifiable,
    ): array {
        // Custom query logic (e.g., pagination, filtering by type)
        return parent::forNotifiable($notifiable);
    }
}
```

## API Reference

### NotificationRepositoryInterface

| Method | Description |
|---|---|
| `forNotifiable(NotifiableInterface $notifiable): array` | Get all notifications for a notifiable, most recent first. Returns `array<DatabaseNotification>`. |
| `unread(NotifiableInterface $notifiable): array` | Get all unread notifications for a notifiable, most recent first. Returns `array<DatabaseNotification>`. |
| `markAsReadFor(NotifiableInterface $notifiable, string $notificationId): bool` | Mark a single notification as read only if it belongs to the notifiable. Returns `false` when it does not exist or belongs to someone else. The default for request-driven code. |
| `markAsRead(string $notificationId): void` | Mark a single notification as read by ID alone, whoever owns it. Admin and internal use only. |
| `markAllAsRead(NotifiableInterface $notifiable): void` | Mark all notifications as read for a notifiable. |
| `deleteFor(NotifiableInterface $notifiable, string $notificationId): bool` | Delete a single notification only if it belongs to the notifiable. Returns `false` when nothing was deleted. The default for request-driven code. |
| `delete(string $notificationId): void` | Delete a single notification by ID alone, whoever owns it. Admin and internal use only. |
| `deleteAll(NotifiableInterface $notifiable): void` | Delete all notifications for a notifiable. |
| `unreadCount(NotifiableInterface $notifiable): int` | Count unread notifications for a notifiable. |

### DatabaseNotification

Entity mapped to the `notifications` table.

| Property | Type | Description |
|---|---|---|
| `$id` | `string` | UUID primary key (varchar 36). |
| `$type` | `string` | Notification class name. |
| `$notifiableType` | `string` | Notifiable entity class name. |
| `$notifiableId` | `string` | Notifiable entity ID. |
| `$data` | `string` | JSON-encoded notification data. |
| `$readAt` | `?string` | Timestamp when read (database timezone), or `null` if unread. |
| `$createdAt` | `string` | Timestamp when the notification was created (database timezone). |
