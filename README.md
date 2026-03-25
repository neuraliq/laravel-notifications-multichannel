# Laravel Multi-Channel Notifications Skill

An AI agent skill for sending notifications across mail, database, broadcast, Slack, and SMS channels.

## Topics Covered

- Multi-channel notification classes (mail, database, broadcast, Slack)
- Dynamic channel routing based on user preferences
- Database notifications with read/unread management
- Notification API endpoints (list, mark read, delete)
- User notification preferences system
- Queued notifications with per-channel queue config
- On-demand notifications for non-user recipients

## Installation

```bash
npx skills add neuraliq/laravel-notifications-multichannel
# or
php artisan boost:add-skill neuraliq/laravel-notifications-multichannel
```

## Compatibility

Claude Code, Cursor, Windsurf, GitHub Copilot

## Requirements

Laravel 11+, Redis (for queued notifications)

## License

MIT
