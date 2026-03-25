# Notification Testing

## Fake Notifications

```php
use Illuminate\Support\Facades\Notification;

it('sends order shipped notification', function () {
    Notification::fake();

    $user = User::factory()->create();
    $order = Order::factory()->create(['user_id' => $user->id]);

    $user->notify(new OrderShipped($order));

    Notification::assertSentTo($user, OrderShipped::class);
});

it('sends to correct channels', function () {
    Notification::fake();

    $user = User::factory()->create();
    $order = Order::factory()->create(['user_id' => $user->id, 'total' => 1500]);

    $user->notify(new OrderShipped($order));

    Notification::assertSentTo($user, OrderShipped::class, function ($notification, $channels) {
        return in_array('mail', $channels)
            && in_array('database', $channels)
            && in_array('slack', $channels); // total > 1000
    });
});
```

## Assert Not Sent

```php
it('does not notify opted-out users', function () {
    Notification::fake();

    $user = User::factory()->create();
    $user->notificationPreferences()->create([
        'channel' => 'mail',
        'type' => 'order_shipped',
        'enabled' => false,
    ]);

    $user->notify(new OrderShipped($order));

    Notification::assertNotSentTo($user, OrderShipped::class, function ($n, $channels) {
        return in_array('mail', $channels);
    });
});
```

## Database Notification Testing

```php
it('stores notification in database', function () {
    $user = User::factory()->create();
    $order = Order::factory()->create(['user_id' => $user->id]);

    $user->notify(new OrderShipped($order));

    expect($user->notifications)->toHaveCount(1);
    expect($user->unreadNotifications->first()->data)
        ->toHaveKey('order_id', $order->id)
        ->toHaveKey('url');
});

it('marks notification as read', function () {
    $user = User::factory()->create();
    $user->notify(new OrderShipped($order));

    $user->unreadNotifications->first()->markAsRead();

    expect($user->unreadNotifications)->toHaveCount(0);
    expect($user->readNotifications)->toHaveCount(1);
});
```
