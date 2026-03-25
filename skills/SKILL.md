---
name: laravel-notifications-multichannel
description: "Laravel multi-channel notifications — mail, database, broadcast, Slack, SMS, and custom channels. Use when creating notification classes, sending via multiple channels, building notification preferences, implementing database notifications with read/unread, on-demand notifications, queued notifications, or markdown mail notifications. Triggers on tasks involving notification creation, channel routing, NotificationSent events, notification tables, broadcast notifications, Slack webhooks, or any notification system design. Use PROACTIVELY whenever notifications, alerts, emails to users, or messaging systems are mentioned."
compatible_agents:
  - Claude Code
  - Cursor
  - Windsurf
  - Copilot
tags:
  - laravel
  - notifications
  - email
  - sms
  - slack
  - broadcast
  - database
  - php
---

# Laravel Multi-Channel Notifications

Send notifications across mail, database, broadcast (WebSocket), Slack, SMS, and custom channels from a single notification class.

## Creating Notifications

```bash
php artisan make:notification OrderShipped
```

### Multi-Channel Notification

```php
namespace App\Notifications;

use App\Models\Order;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Messages\MailMessage;
use Illuminate\Notifications\Messages\BroadcastMessage;
use Illuminate\Notifications\Messages\SlackMessage;
use Illuminate\Notifications\Notification;

class OrderShipped extends Notification implements ShouldQueue
{
    use Queueable;

    public function __construct(
        public Order $order
    ) {}

    public function via(object $notifiable): array
    {
        $channels = ['database'];

        // Always email
        $channels[] = 'mail';

        // Broadcast if user is likely online
        if ($notifiable->last_active_at?->isAfter(now()->subMinutes(5))) {
            $channels[] = 'broadcast';
        }

        // Slack for high-value orders
        if ($this->order->total > 1000) {
            $channels[] = 'slack';
        }

        // Respect user preferences
        return array_filter($channels, fn ($ch) =>
            $notifiable->wantsNotification($ch)
        );
    }

    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
            ->subject("Order #{$this->order->number} Shipped")
            ->greeting("Hello {$notifiable->name}!")
            ->line("Your order #{$this->order->number} has been shipped.")
            ->line("Tracking: {$this->order->tracking_number}")
            ->action('Track Order', url("/orders/{$this->order->id}"))
            ->line('Thank you for your purchase!');
    }

    public function toDatabase(object $notifiable): array
    {
        return [
            'order_id' => $this->order->id,
            'order_number' => $this->order->number,
            'message' => "Order #{$this->order->number} has been shipped.",
            'tracking_number' => $this->order->tracking_number,
            'url' => "/orders/{$this->order->id}",
        ];
    }

    public function toBroadcast(object $notifiable): BroadcastMessage
    {
        return new BroadcastMessage([
            'order_id' => $this->order->id,
            'message' => "Order #{$this->order->number} shipped!",
            'url' => "/orders/{$this->order->id}",
        ]);
    }

    public function toSlack(object $notifiable): SlackMessage
    {
        return (new SlackMessage)
            ->success()
            ->content("Order #{$this->order->number} shipped — \${$this->order->total}");
    }
}
```

### Sending

```php
// To a single user
$user->notify(new OrderShipped($order));

// To multiple users
Notification::send($admins, new OrderShipped($order));

// On-demand (no User model needed)
Notification::route('mail', 'guest@example.com')
    ->route('slack', '#orders')
    ->notify(new OrderShipped($order));
```

## Database Notifications

```bash
php artisan notifications:table
php artisan migrate
```

```php
// User model
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use Notifiable;
}

// Read notifications
$user->notifications;              // All
$user->unreadNotifications;        // Unread only
$user->readNotifications;          // Read only
$user->unreadNotifications->count(); // Unread count

// Mark as read
$notification->markAsRead();
$user->unreadNotifications->markAsRead(); // Mark all
```

### API Endpoints

```php
// app/Http/Controllers/NotificationController.php
class NotificationController extends Controller
{
    public function index(Request $request)
    {
        return $request->user()
            ->notifications()
            ->paginate(20);
    }

    public function markAsRead(Request $request, string $id)
    {
        $request->user()
            ->notifications()
            ->findOrFail($id)
            ->markAsRead();

        return response()->noContent();
    }

    public function markAllAsRead(Request $request)
    {
        $request->user()->unreadNotifications->markAsRead();

        return response()->noContent();
    }

    public function destroy(Request $request, string $id)
    {
        $request->user()
            ->notifications()
            ->findOrFail($id)
            ->delete();

        return response()->noContent();
    }
}
```

## User Notification Preferences

```php
// migration
Schema::create('notification_preferences', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->string('channel');   // mail, database, broadcast, slack
    $table->string('type');      // order_shipped, payment_received, etc.
    $table->boolean('enabled')->default(true);
    $table->timestamps();
    $table->unique(['user_id', 'channel', 'type']);
});

// User model
public function wantsNotification(string $channel, ?string $type = null): bool
{
    if (! $type) return true;

    $pref = $this->notificationPreferences()
        ->where('channel', $channel)
        ->where('type', $type)
        ->first();

    return $pref?->enabled ?? true; // Default: enabled
}
```

## Queued Notifications

```php
class OrderShipped extends Notification implements ShouldQueue
{
    use Queueable;

    public $queue = 'notifications';
    public $delay = 60; // seconds
    public $tries = 3;
    public $backoff = [30, 120, 600];

    // Per-channel queue
    public function viaQueues(): array
    {
        return [
            'mail' => 'notifications',
            'slack' => 'notifications',
            'database' => 'sync', // Immediate
        ];
    }
}
```

## Key Rules

1. Always implement `ShouldQueue` on notifications — sending email synchronously blocks requests
2. Always implement `toDatabase()` as a baseline — users expect a notification center
3. Always include a `url` field in database notifications — links to the relevant resource
4. Always respect user preferences in `via()` — never force channels users opted out of
5. Always use `on-demand` notifications for non-user recipients (guest emails, Slack channels)
6. Always set per-channel queues — database can be sync, mail should be queued
7. Always include `ShouldQueue` retry logic — `$tries`, `$backoff` for transient failures
8. Use broadcast for real-time UI updates — pair with database for persistence
9. Always clean old notifications — schedule `DB::table('notifications')->where('created_at', '<', now()->subMonths(3))->delete()`
10. Test with `Notification::fake()` — assert channels, recipients, and data without sending
