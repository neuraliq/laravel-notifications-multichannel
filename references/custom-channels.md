# Custom Notification Channels

## Creating a Custom Channel

```php
namespace App\Channels;

use Illuminate\Notifications\Notification;

class TelegramChannel
{
    public function send(object $notifiable, Notification $notification): void
    {
        $message = $notification->toTelegram($notifiable);
        $chatId = $notifiable->routeNotificationFor('telegram');

        if (! $chatId) return;

        Http::post("https://api.telegram.org/bot" . config('services.telegram.bot_token') . "/sendMessage", [
            'chat_id' => $chatId,
            'text' => $message['text'],
            'parse_mode' => 'HTML',
        ]);
    }
}
```

## Using in Notification

```php
public function via(object $notifiable): array
{
    return ['database', 'mail', TelegramChannel::class];
}

public function toTelegram(object $notifiable): array
{
    return [
        'text' => "<b>Order #{$this->order->number}</b>\nStatus: Shipped\nTracking: {$this->order->tracking_number}",
    ];
}
```

## Route for Custom Channel

```php
// User model
public function routeNotificationForTelegram(): ?string
{
    return $this->telegram_chat_id;
}
```

## Markdown Mail Notifications

```php
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)
        ->subject('Weekly Report')
        ->markdown('emails.weekly-report', [
            'user' => $notifiable,
            'stats' => $this->stats,
        ]);
}
```

```blade
{{-- resources/views/emails/weekly-report.blade.php --}}
<x-mail::message>
# Weekly Report

Hello {{ $user->name }},

Here's your summary for this week:

<x-mail::table>
| Metric | Value |
|--------|-------|
| Orders | {{ $stats['orders'] }} |
| Revenue | ${{ number_format($stats['revenue'], 2) }} |
</x-mail::table>

<x-mail::button :url="route('dashboard')">
View Dashboard
</x-mail::button>

Thanks,<br>
{{ config('app.name') }}
</x-mail::message>
```

## Rate Limiting Notifications

```php
use Illuminate\Cache\RateLimiting\Limit;

class AlertNotification extends Notification implements ShouldQueue
{
    public function shouldSend(object $notifiable, string $channel): bool
    {
        // Max 3 alerts per hour per user
        $key = "notification:{$notifiable->id}:alert";

        if (Cache::get($key, 0) >= 3) {
            return false;
        }

        Cache::increment($key);
        Cache::put($key, Cache::get($key), 3600);

        return true;
    }
}
```
