# Structured Logging

The logging system provides file-based, daily-rotated logs with level filtering and structured context data.

<meta name="description" content="Learn how to use WebFiori's structured logging with FileLogger, levels, and context.">

## Basic Usage

```php
use WebFiori\Log\FileLogger;

$logger = new FileLogger(APP_PATH . 'Storage' . DS . 'Logs');

$logger->info('Order created', ['order_id' => 42, 'total' => 99.99]);
$logger->error('Payment failed', ['order_id' => 42, 'error' => 'Card declined']);
$logger->debug('Cache hit', ['key' => 'posts:page:1']);
```

Output in `App/Storage/Logs/app-2026-05-31.log`:

```
[2026-05-31 14:30:00] [INFO] Order created {"order_id":42,"total":99.99}
[2026-05-31 14:30:01] [ERROR] Payment failed {"order_id":42,"error":"Card declined"}
[2026-05-31 14:30:02] [DEBUG] Cache hit {"key":"posts:page:1"}
```

## Log Levels

| Level | Use for |
|-------|---------|
| `debug` | Detailed diagnostic information |
| `info` | Normal operations (order placed, email sent) |
| `warning` | Unexpected but non-critical situations |
| `error` | Failures that need attention |
| `critical` | System-level failures (DB down, disk full) |

## Level Filtering

Set a minimum level to suppress lower-priority messages:

```php
use WebFiori\Log\FileLogger;
use WebFiori\Log\LogLevel;

// Only log warnings and above (skip debug and info)
$logger = new FileLogger(APP_PATH . 'Storage/Logs', LogLevel::WARNING);
```

## Available Methods

```php
$logger->debug($message, $context);
$logger->info($message, $context);
$logger->warning($message, $context);
$logger->error($message, $context);
$logger->critical($message, $context);
```

All methods accept an optional `$context` array that is JSON-encoded in the log entry.

## Daily Rotation

Log files are automatically named by date: `app-YYYY-MM-DD.log`. A new file is created each day. Old files remain until you clean them up (via a background task or external tool).

## Custom Logger

Implement the `Logger` interface for alternative backends:

```php
use WebFiori\Log\Logger;

class DatabaseLogger implements Logger {
    public function debug(string $message, array $context = []): void { /* ... */ }
    public function info(string $message, array $context = []): void { /* ... */ }
    public function warning(string $message, array $context = []): void { /* ... */ }
    public function error(string $message, array $context = []): void { /* ... */ }
    public function critical(string $message, array $context = []): void { /* ... */ }
}
```

## Best Practices

- Use context arrays for structured data — don't interpolate into the message string
- Set `LogLevel::INFO` in production, `LogLevel::DEBUG` in development
- Include correlation IDs (order_id, user_id, request_id) for traceability
- Log at boundaries: incoming requests, outgoing calls, state changes
- Clean up old log files with a scheduled task
